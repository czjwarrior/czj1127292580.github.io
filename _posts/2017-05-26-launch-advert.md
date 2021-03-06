---
layout: post
title: "iOS广告启动页"
date: 2017-05-26
tag: iOS
---
产品需求：启动页逻辑: 上部分为广告区域，可在运营后台配置图片+跳转页(同No.2);首次开 APP，则请求一次配置，失败或无配置则不显示，一旦有网了即刻请求一次并 做好缓存; 客户端每4小时请求一次;每两小时显示一次广告页内容 用户点击则跳转已配置页面;启动页上有5s倒计时，时间到了启动页关闭，也 可手动点击跳过启动页，广告已过期也不显示。

服务端返回模型： 
{ 
“image” : “http://www.baidu.com“, 
“link” : “跳转链接”, 
“start” : “2017-05-24 08:00:00”, 
“end” : “2017-05-31 23:59:59” 
}

- 解决方案一、 接口请求成功后，将图片缓存到本地路径，同时保存有效期等字段 

- 解决方案二、 将整个模型保存到NSUserDefauts，将下载到的图片也保存到NSUserDefauts，给模型加一个NSData类型字段，（因为UIImage类型不能直接保存） 
为了方便，直接用方案二：

1、为方便管理，建立一个Manager类 YDWAdvertManager

```
.h文件
```

```
#import <Foundation/Foundation.h>
#import "YDWAdvertModel.h"
#import "YDWAdvertView.h"

#define  YDWAdvertMgrInit YDWAdvertMgr
#define  YDWAdvertMgr [YDWAdvertManager sharedYDWAdvertManager]

@interface YDWAdvertManager : NSObject

@property (nonatomic, strong, readonly) YDWAdvertModel *advertModel;

+ (YDWAdvertManager *)sharedYDWAdvertManager;

// 刷新广告
- (void)refreshAdvert;

// 是否有有效的cache
- (BOOL)hasValidLocalCache;

// 显示广告
- (void)showAdFinished:(YDWAdvertViewDidFinishedBlock)finishedBlock;
- (void)showAdInView:(UIView *)aView finished:(YDWAdvertViewDidFinishedBlock)finishedBlock;

@end
```

```
.m文件
```

```
static NSTimeInterval kYDWAdvertMinFetchTime = 4 * 60 * 60;  // 刷新最小时间间隔

@interface YDWAdvertManager ()

@property (nonatomic, strong, readwrite) YDWAdvertModel *advertModel;
@property (nonatomic, assign) BOOL downloading;

@property (nonatomic, assign) NSTimeInterval lastFetchTime;

@end

@implementation YDWAdvertManager
SYNTHESIZE_SINGLETON_FOR_CLASS(YDWAdvertManager)

- (instancetype)init{
    if (self = [super init]) {
        [self setDownloading:NO];
        
        NSData *cacheData = [ZQHelper getFromUserDefaultsWithKey:kYDWKeyUserDefaultAdvertModel];
        if (cacheData) {
           self.advertModel = [NSKeyedUnarchiver unarchiveObjectWithData:cacheData];
        }

        // 每次启动的时候加载图片
        [self refreshAdvert];
        
        @weakify(self)
        [[[NSNotificationCenter defaultCenter] rac_addObserverForName:UIApplicationWillEnterForegroundNotification object:nil] subscribeNext:^(id x) {
            @strongify(self)
            [self refreshAdvert];
        }];
        
        // 监听后台轮训
        [[YDWHomeMonitor rac_signalForSelector:@selector(updateData)] subscribeNext:^(id x) {
            @strongify(self)
            [self refreshAdvert];
        }];
    }
    return self;
}

#pragma mark - Private

// 下载广告图片
- (void)downloadAdImageWithAdvertModel:(YDWAdvertModel *)aAdvert{
    
    if (!aAdvert || !aAdvert.image) {
        [self setDownloading:NO];
        return;
    }
    
    @weakify(self)
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        @strongify(self)
        NSData *data = [NSData dataWithContentsOfURL:[NSURL URLWithString:aAdvert.image]];
        
        [aAdvert setCachedImageData:data];

        dispatch_async(dispatch_get_main_queue(), ^{
        
            NSData *cacheData = [NSKeyedArchiver archivedDataWithRootObject:aAdvert];
            [ZQHelper storeToUserDefaultsWithValue:cacheData andKey:kYDWKeyUserDefaultAdvertModel];
            
            self.advertModel = aAdvert;
            
            [self setDownloading:NO];
        });
    });
}

#pragma mark - Public

// 刷新广告
- (void)refreshAdvert{
    if (self.downloading) {return;}
    
    @weakify(self)
    
    // 判断距离上一次刷新是否超过4小时
    NSTimeInterval currentTimeInterval = [[NSDate date] timeIntervalSince1970];
    if (currentTimeInterval - self.lastFetchTime <= kYDWAdvertMinFetchTime) {
        return;
    }
    
    self.lastFetchTime = currentTimeInterval;
    
    [self setDownloading:YES];
    
//    // 请求广告接口
    YDWAdvertApi *api = [[YDWAdvertApi alloc] init];
    [api startWithCompletionBlockWithSuccess:^(SSBaseRequest *request) {
        @strongify(self)
        
        YDWAdvertApi *mRequest = (YDWAdvertApi *)request;
        
        YDWAdvertModel *advertModel = [mRequest getAdvertData];
        
        if (![self.advertModel.image isEqualToString:advertModel.image] || !self.advertModel.cachedImageData) {
            [self downloadAdImageWithAdvertModel:advertModel];
            
        } else {
            [self setDownloading:NO];
        }
        
    } failure:^(SSBaseRequest *request) {
        @strongify(self)
        [self setDownloading:NO];
    }];
}

// 是否有有效的cache
- (BOOL)hasValidLocalCache{
    
    if (!self.advertModel) {return NO;}
    if (!self.advertModel.isExpired) {return NO;}
    if (!self.advertModel.cachedImageData) {return NO;}
    
    return YES;
}

// 显示广告
- (void)showAdFinished:(YDWAdvertViewDidFinishedBlock)finishedBlock{
    [self showAdInView:nil finished:finishedBlock];
}

- (void)showAdInView:(UIView *)aView finished:(YDWAdvertViewDidFinishedBlock)finishedBlock{
    if (!aView) {
        aView = [UIApplication sharedApplication].keyWindow;
    }
    
    [YDWAdvertView showInView:aView
                        image:[UIImage imageWithData:self.advertModel.cachedImageData]
                finishedBlock:finishedBlock];
}
```

2、UI文件

```
#import "YDWAdvertView.h"
#import "YDBrowserViewModel.h"
#import "YDWAdvertManager.h"

// 广告显示的时间
static int const showtime = 5;

@interface YDWAdvertView()

@property (nonatomic, strong) UIImageView   *adImageView;
@property (nonatomic, strong) UIView        *bottomLogoView;

@property (nonatomic, strong) UIImageView   *logoImageVIew;

@property (nonatomic, strong) UIButton      *skipBtn;

@property (nonatomic, strong) NSTimer       *countTimer;

@property (nonatomic, assign) int count;

@property (nonatomic, copy) YDWAdvertViewDidFinishedBlock finishedBlock;

@end

@implementation YDWAdvertView

- (instancetype)initWithFrame:(CGRect)frame
{
    if (self = [super initWithFrame:frame]) {
        
        [self setBackgroundColor:SS_WHITE_COLOR];
        [self addSubview:self.adImageView];
        [self addSubview:self.bottomLogoView];
        [self addSubview:self.skipBtn];
        
        @weakify(self)
        [self.adImageView setTapActionWithBlock:^{
           @strongify(self)
            [self dismissToAdPage:YES];
            [YDBrowserViewModel openUrl:YDWAdvertMgr.advertModel.link title:@""];
        }];
        
        [[self.skipBtn rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(id x) {
            @strongify(self)
            [self dismissToAdPage:NO];
        }];
    }
    return self;
}

#pragma mark - Getter

- (UIButton *)skipBtn{
    if (_skipBtn == nil) {
        _skipBtn = [UIButton createBtnByTitle:@"跳过 5s"
                                   titleColor:SS_WHITE_COLOR
                                     textFont:SS_NORMAL_FONT_WITH_6P(14, 17)];
        [_skipBtn setBackgroundColor:[SS_BLACK_COLOR colorWithAlphaComponent:.2f] forState:UIControlStateNormal];
        [_skipBtn setView_size:SS_ADAPT_SCALE_FLOAT_SIZE_6P(64, 24, 82, 30)];
    }
    return _skipBtn;
}

- (UIImageView *)adImageView{
    if (_adImageView == nil) {
        _adImageView = [UIImageView new];
        [_adImageView setUserInteractionEnabled:YES];
        [_adImageView setContentMode:UIViewContentModeScaleAspectFill];
        [_adImageView setView_size:CGSizeMake(SS_SCREEN_WIDTH, SS_SCREEN_WIDTH * 4 / 3)];
    }
    return _adImageView;
}

- (UIView *)bottomLogoView{
    if (_bottomLogoView == nil) {
        _bottomLogoView = [[UIView alloc] init];
        [_bottomLogoView setView_size:CGSizeMake(SS_SCREEN_WIDTH, self.view_height - self.adImageView.view_height)];
        [_bottomLogoView addSubview:self.logoImageVIew];
    }
    return _bottomLogoView;
}

- (UIImageView *)logoImageVIew{
    if (_logoImageVIew == nil) {
        _logoImageVIew = [UIImageView createByImageName:@"ydw_lanuch_ad_logo"];
        
    }
    return _logoImageVIew;
}

- (NSTimer *)countTimer{
    if (!_countTimer) {
        _countTimer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                       target:self
                                                     selector:@selector(countDown:)
                                                     userInfo:nil
                                                      repeats:YES];
    }
    return _countTimer;
}

#pragma mark - Super

- (void)layoutSubviews{
    [super layoutSubviews];
    
    [self.adImageView setView_minX:0];
    [self.adImageView setView_minY:0];
    [self.adImageView setFrameValuesToInt];
    
    [self.bottomLogoView setView_minX:0];
    [self.bottomLogoView setView_minY:self.adImageView.view_maxY];
    [self.bottomLogoView setFrameValuesToInt];
    
    [self.logoImageVIew sizeToFit];
    [self.logoImageVIew setView_centerX:self.bottomLogoView.centerOfView.x];
    [self.logoImageVIew setView_centerY:self.bottomLogoView.centerOfView.y];
    [self.logoImageVIew setFrameValuesToInt];
    
    [self.skipBtn setView_maxX:SS_SCREEN_WIDTH - SS_ADAPT_SCALE_FLOAT_WITH_6P(15, 20)];
    [self.skipBtn setView_minY:SS_ADAPT_SCALE_FLOAT_WITH_6P(15, 20)];
    [self.skipBtn setFrameValuesToInt];
}

#pragma mark - Setter

- (void)setShowImage:(UIImage *)showImage{
    [self.adImageView setImage:showImage];
}

#pragma mark - UI Util

- (CGFloat)logoImageViewHeight{
    return SS_ADAPT_SCALE_FLOAT_WITH_6P(143, 184);
}

#pragma mark - Private

- (void)showInView:(UIView *)aView{
    if (!aView) {
        aView = [UIApplication sharedApplication].keyWindow;
    }
    [aView addSubview:self];
    
    [self startTimer];
}

// 定时器倒计时
- (void)startTimer{
    _count = showtime;
    [[NSRunLoop mainRunLoop] addTimer:self.countTimer forMode:NSRunLoopCommonModes];
}

// 移除广告页面
- (void)dismissToAdPage:(BOOL)toAdPage{
    [self.countTimer invalidate];
    [self setCountTimer:nil];
    
    if (self.finishedBlock) {
        self.finishedBlock(toAdPage);
    }
    
    [UIView animateWithDuration:0.3f animations:^{
        self.alpha = 0.f;
        
    } completion:^(BOOL finished) {
        [self removeFromSuperview];
        
    }];
}

#pragma mark - Target

- (void)countDown:(NSTimer *)aTimer{
    _count--;
    [self.skipBtn setTitle:[NSString stringWithFormat:@"跳过 %ds",_count] forState:UIControlStateNormal];
    if (_count == 0) {
        [self dismissToAdPage:NO];
    }
}

#pragma mark - Public

+ (instancetype)showInView:(UIView *)aView
                     image:(UIImage *)aImage
             finishedBlock:(YDWAdvertViewDidFinishedBlock)finishedBlock{
   
    YDWAdvertView *advertView = [[YDWAdvertView alloc] initWithFrame:aView.bounds];
    [advertView setShowImage:aImage];
    [advertView setFinishedBlock:finishedBlock];
    
    [advertView showInView:aView];
    
    return advertView;
}
```

3、外部调用

```
NSDate *date = [ZQHelper getFromUserDefaultsWithKey:kYDWKeyUserDefaultAdLaunch];
            NSInteger hours = [date hoursBeforeDate:[NSDate date]];
            
            if (([YDWAdvertMgr hasValidLocalCache] && hours >= 2) || ([YDWAdvertMgr hasValidLocalCache] && !date)) {
                
                [ZQHelper storeToUserDefaultsWithValue:[NSDate date] andKey:kYDWKeyUserDefaultAdLaunch];
                
                [YDWAdvertMgr showAdInView:[UIApplication sharedApplication].keyWindow finished:^(BOOL toAdPage) {
                    if (toAdPage) {
                        [self stepIntoAppFromAd];
                        
                    } else {
                        [self stepIntoApp];
                    }
                }];
                
            } else {
                [self stepIntoApp];
            }
```



