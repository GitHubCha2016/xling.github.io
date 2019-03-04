
AVCaptureSession是AVFoundation的核心类,用于捕捉视频和音频,协调视频和音频的输入和输出流.
- 输入源
设备AVCaptureDevice
[AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo]
- 输出源
添加AVCaptureOutput,即AVCaptureSession的输出源.一般输出源分成:音视频源,图片源,文件源等.
	音视频输出AVCaptureAudioDataOutput,AVCaptureVideoDataOutput.
	静态图片输出AVCaptureStillImageOutput(iOS10中被AVCapturePhotoOutput取代了)
	AVCaptureMovieFileOutput表示文件源.
### 创建会话AVCaptureSession
```
-(AVCaptureSession *)captureSession{
	if (!_captureSession) {
		_captureSession = [[AVCaptureSession alloc]init];
		if ([[UIDevice currentDevice]userInterfaceIdiom] == UIUserInterfaceIdiomPhone) {
			customLog(@"当前使用的是手机");
		}
	}
	return _captureSession;
}
```
### 创建输入源
**1、创建AVCaptureDevice**
```
-(AVCaptureDevice *)getCaptureDevice{
	// 每次获取都要重新创建
	NSArray * devices = [AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo];
	for (AVCaptureDevice * device in devices) {
		if (device.position == self.captureDevicePosition) {
			NSError * error = nil;
			if ([device lockForConfiguration:&error]) {
				// 焦点 曝光 黑白色 -> 自动
				if ([device isFocusModeSupported:AVCaptureFocusModeContinuousAutoFocus]) {
					device.focusMode = AVCaptureFocusModeContinuousAutoFocus;
				}
				if ([device isExposureModeSupported:AVCaptureExposureModeContinuousAutoExposure]) {
					device.exposureMode = AVCaptureExposureModeContinuousAutoExposure;
				}
				if ([device isWhiteBalanceModeSupported:AVCaptureWhiteBalanceModeContinuousAutoWhiteBalance]) {
					device.whiteBalanceMode = AVCaptureWhiteBalanceModeContinuousAutoWhiteBalance;
				}
				
				[device unlockForConfiguration];
			}
			
			_captureDevice = device;
			return _captureDevice;
		}
	}
	return nil;
}
```
**2、创建AVCaptureDeviceInput**
```
-(AVCaptureDeviceInput *)getCaptureDeviceInput{
	NSError * error = nil;
	AVCaptureDevice * device = [self getCaptureDevice];
	_captureDeviceInput = [AVCaptureDeviceInput deviceInputWithDevice:device error:&error];
	if (error || device == nil) {
		NSLog(@"创建失败: %@",error.description);
	}
	return _captureDeviceInput;
}
```
**3、添加输入源到会话中**

```
AVCaptureSession * session = self.captureSession;
	// 添加输入对象到会话
	if ([session.inputs containsObject:self.captureDeviceInput]) {
		[session removeInput:self.captureDeviceInput];
	}
	AVCaptureDeviceInput * input = [self getCaptureDeviceInput];
	if ([session canAddInput:input]) {
		[session addInput:input];
	}
```
### 创建输出源并添加到会话中
```
 // 添加照片输出
	if (SYSTEM_VERSION_GREATER_THAN_OR_EQUAL_TO(@"10.0")) {
		AVCapturePhotoOutput * photoOutput = [[AVCapturePhotoOutput alloc]init];
		
		// 先移除
		if ([session.outputs containsObject:self.captureOutput]) {
			[session removeOutput:self.captureOutput];
		}
		// 添加
		if ([session canAddOutput:photoOutput]) {
			[session addOutput:photoOutput];
		}
		
		self.captureOutput = photoOutput;
	}else{
		AVCaptureStillImageOutput * stillImageOutput = [[AVCaptureStillImageOutput alloc]init];
		NSDictionary *myOutputSettings = [[NSDictionary alloc] initWithObjectsAndKeys:AVVideoCodecJPEG,AVVideoCodecKey,nil];
		[stillImageOutput setOutputSettings:myOutputSettings];
		
		// 先移除
		if ([session.outputs containsObject:self.captureOutput]) {
			[session removeOutput:self.captureOutput];
		}
		// 添加
		if ([session canAddOutput:stillImageOutput]){
			[session addOutput:stillImageOutput];
		}
		
		self.captureOutput = stillImageOutput;
	}	
```
输出对象-视频流对象作为输出适合及时获取输出数据(直播)
```
	AVCaptureVideoDataOutput *videoDataOutput = [[AVCaptureVideoDataOutput alloc] init];
	[videoDataOutput setAlwaysDiscardsLateVideoFrames:YES];// 默认为YES
	// 设置代理，捕获视频样品数据
	// 注意：队列必须是串行队列，才能获取到数据，而且不能为空
	dispatch_queue_t photoCaptureQueue = dispatch_queue_create(MDPhotoCaptureQueue, DISPATCH_QUEUE_SERIAL);
	[videoDataOutput setSampleBufferDelegate:self queue:photoCaptureQueue];
	if ([session canAddOutput:videoDataOutput]) {
		[session addOutput:videoDataOutput];
	}
	#pragma mark AVCaptureVideoDataOutputSampleBufferDelegate
- (void)captureOutput:(AVCaptureOutput *)output didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection{
- // 及时获取视频输出数据
	customLog(@"采集到数据");
}
```
### 添加预览图层

```
-(void)addPreviewLayer{
	// 视频预览图层
	AVCaptureVideoPreviewLayer *previedLayer = [AVCaptureVideoPreviewLayer layerWithSession:self.captureSession];
	previedLayer.frame = [UIScreen mainScreen].bounds;
	_previewLayer.videoGravity = AVLayerVideoGravityResizeAspectFill;
	[self.view.layer insertSublayer:previedLayer atIndex:0];
	_previewLayer = previedLayer;
}
```
### 切换前后置摄像头-切换输入源
```
- (void)changeInput{
	// 移除输入对象
	if (self.captureDeviceInput && [self.captureSession.inputs containsObject:self.captureDeviceInput]) {
		[self.captureSession removeInput:self.captureDeviceInput];
	}
	// 添加输入对象到会话
	AVCaptureDeviceInput * input = [self getCaptureDeviceInput];
	if ([self.captureSession canAddInput:input]) {
		[self.captureSession addInput:input];
	}
	self.isChangeCaptureDevicePostion = NO;
	// 给摄像头的切换添加翻转动画
	CATransition *animation = [CATransition animation];
	animation.duration = .5f;
	animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
	animation.type = @"oglFlip";
	if (self.captureDevicePosition == AVCaptureDevicePositionFront) {
		animation.subtype = kCATransitionFromLeft;
	}else{
		animation.subtype = kCATransitionFromRight;
	}
	[self.previewLayer addAnimation:animation forKey:nil];
}
```
### 拍照按钮点击-获取图片
```-
-(void)commitButtonAction{
	// 拍照
	if (SYSTEM_VERSION_GREATER_THAN_OR_EQUAL_TO(@"10.0")) {
		
		AVCapturePhotoOutput * output = (AVCapturePhotoOutput *)self.captureOutput;
		
		//从 AVCaptureStillImageOutput 中取得 AVCaptureConnection
		self.captureConnection = [output connectionWithMediaType:AVMediaTypeVideo];
		AVCapturePhotoSettings * settings = [AVCapturePhotoSettings photoSettings];
		
		[output capturePhotoWithSettings:settings delegate:self];
	}else{
		AVCaptureStillImageOutput * stillImageOutput = (AVCaptureStillImageOutput *)self.captureOutput;
		//从 AVCaptureStillImageOutput 中取得 AVCaptureConnection
		self.captureConnection = [stillImageOutput connectionWithMediaType:AVMediaTypeVideo];
		[stillImageOutput captureStillImageAsynchronouslyFromConnection:self.captureConnection completionHandler:^(CMSampleBufferRef  _Nullable imageDataSampleBuffer, NSError * _Nullable error) {
			if (imageDataSampleBuffer != nil) {
				NSData * data = [AVCaptureStillImageOutput jpegStillImageNSDataRepresentation:imageDataSampleBuffer];
				// 取得静态影像 两种获取方式
				//UIImage * image = [[UIImage alloc]initWithData:data];
				
				CFDataRef cfData = CFBridgingRetain(data);
				CGDataProviderRef dataProvider = CGDataProviderCreateWithCFData(cfData);
				CGImageRef cgImage = CGImageCreateWithJPEGDataProvider(dataProvider, nil, YES, kCGRenderingIntentDefault);
				CFBridgingRelease(cfData);
				
				// 裁剪图片
				[self cropWithImage:[UIImage imageWithCGImage:cgImage]];

				// 取得影像数据（需要ImageIO.framework 与 CoreMedia.framework）
				CFDictionaryRef attachments = CMGetAttachment(imageDataSampleBuffer, kCGImagePropertyExifDictionary, NULL);
				customLog(@"影像属性: %@", attachments);
			}
		}];
	}
}
```
### 代理获取图片AVCapturePhotoOutput
```
#pragma mark AVCapturePhotoCaptureDelegate
// IOS - 11
- (void)captureOutput:(AVCapturePhotoOutput *)output didFinishProcessingPhoto:(AVCapturePhoto *)photo error:(NSError *)error{
	if (error) {
		customLog(@"获取图片错误 --- %@",error.localizedDescription);
	}
	if (photo) {
		if (@available(iOS 11.0, *)) {
			CGImageRef cgImage = [photo CGImageRepresentation];
			UIImage * image = [UIImage imageWithCGImage:cgImage];
			customLog(@"获取图片成功 --- %@",image);
		} else {
			customLog(@"不是走这个代理方法");
		}
	}
}

- (void)captureOutput:(AVCapturePhotoOutput *)output didFinishProcessingPhotoSampleBuffer:(nullable CMSampleBufferRef)photoSampleBuffer previewPhotoSampleBuffer:(nullable CMSampleBufferRef)previewPhotoSampleBuffer resolvedSettings:(AVCaptureResolvedPhotoSettings *)resolvedSettings bracketSettings:(nullable AVCaptureBracketedStillImageSettings *)bracketSettings error:(nullable NSError *)error{
	if (error) {
		customLog(@"获取图片错误 --- %@",error.localizedDescription);
	}
	if (photoSampleBuffer) {
		NSData *data = [AVCapturePhotoOutput JPEGPhotoDataRepresentationForJPEGSampleBuffer:photoSampleBuffer previewPhotoSampleBuffer:previewPhotoSampleBuffer];
		UIImage *image = [UIImage imageWithData:data];
		customLog(@"获取图片成功 --- %@",image);
	}
}
```
