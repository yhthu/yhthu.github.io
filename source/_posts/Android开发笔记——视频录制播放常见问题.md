title: Android开发笔记——视频录制播放常见问题
tags:
- Android
- 视频录制播放
categories: Android开发笔记

---
本文分享自己在视频录制播放过程中遇到的一些问题，主要包括：

 1. 视频录制流程
 2. 视频预览及SurfaceHolder
 3. 视频清晰度及文件大小
 4. 视频文件旋转


----------
## 一、视频录制流程 ##
以微信为例，其录制触发为按下（住）录制按钮，结束录制的触发条件为松开录制按钮或录制时间结束，其流程大概可以用下图来描述。
![](/img/20151230-1.png)
### 1.1、开始录制
![](/img/20151230-2.png)
初始化过程主要包括View，Data以及Listener三部分。在初始化View时，添加摄像头预览，添加倒计时文本组件，设置初始状态UI组件的可见；初始化Data时，从Intent中获取初始数据；初始化Listener时，分别对录制触发按钮，保存/取消视频录制按钮以及视频预览界面添加监听。
当系统初始化成功后，等待用户按下录制按钮，因此在录制按钮的监听中，需要完成以下功能：录制，计时，更新界面组件。
```
if(isRecording) {
    mMediaRecorder.stop();
    releaseMediaRecorder();
    mCamera.lock();
    isRecording = false;
}
if(startRecordVideo()) {
    startTimeVideoRecord();
    isRecording = true;
}
```
首先判断当前录制状态，如果正在录制，则先停止录制，释放MediaRecorder资源，锁定摄像头，置位录制状态；然后开始视频录制startRecordVideo，其boolean型返回值表征是否启动成功，启动成功后，开始视频录制计时，并且置位录制状态。startRecordVideo涉及MediaRecorder的配置，准备以及启动。
![](/img/20151230-3.png)
翻译成代码如下：
```
private boolean startRecordVideo() {
    configureMediaRecorder();
    if(!prepareConfiguredMediaRecorder()) {
        return false;
    }
    mMediaRecorder.start();
    return true;
}
```
### 1.2、结束录制

根据上述流程图可知，结束录制的触发条件为松开录制按钮或计时时间到。在结束录制方法中，需要释放MediaRecorder，开始循环播放已录制视频，设置界面更新等。
![](/img/20151230-4.png)
 翻译成代码如下：
```
private void stopRecordVideo() {

    releaseMediaRecorder();
    // 录制视频文件处理
    if(currentRecordProgress < MIN_RECORD_TIME) {
        Toast.makeText(VideoInputActivity.this, "录制时间太短", Toast.LENGTH_SHORT).show();
    } else {
        startVideoPlay();
        isPlaying = true;
        setUiDisplayAfterVideoRecordFinish();
    }
    currentRecordProgress = 0;
    updateProgressBar(currentRecordProgress);
    releaseTimer();
    // 状态设置
    isRecording = false;
}
```
## 二、视频预览及SurfaceHolder ##
视频预览采用SurfaceView，相比于普通的View，SurfaceView在一个新起的单独线程中绘制画面，该实现的优点是更新画面不会阻塞UI主线程，缺点是会带来事件同步的问题。当然，这涉及到UI事件的传递以及线程同步，这里不做详细说明，有兴趣的可以参考链接：http://wugengxin.cn/download/pdf/android/PRE_andevcon_mastering-the-android-touch-system.pdf。
在实现中，通过继承SurfaceView组件来实现自定义预览控件。首先，SurfaceView的getHolder()方法会返回SurfaceHolder，需要为SurfaceHolder添加SurfaceHolder.Callback回调；其次，重写surfaceCreated、surfaceChanged和surfaceDestroyed实现。
### 2.1、构造器
构造器包含了初始化域以及添加上述回调的过程。
```
public CameraPreview(Context context, Camera camera) {
    super(context);

    mCamera = camera;
    mSupportedPreviewSizes = mCamera.getParameters().getSupportedPreviewSizes();
    mHolder = getHolder();
    mHolder.addCallback(this);
    mHolder.setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS);
}
```
这里需要说明mSupportedPreviewSizes，由于摄像头支持的预览尺寸由Camera本身的参数决定，因此需要首先获取其所支持的预览尺寸。
### 2.2、预览尺寸的设置
从Google官方的Camera示例程序中可以看出，选择预览尺寸的标准是（1）摄像头支持的预览尺寸的宽高比与SurfaceView的宽高比的绝对差值小于0.1；（2）在（1）获得的尺寸中，选取与SurfaceView的高的差值最小的。通过代码对这两个标准进行了实现，这里贴一下官方的代码：
```
public Camera.Size getOptimalPreviewSize(List<Camera.Size> sizes, int w, int h) {
    final double ASPECT_TOLERANCE = 0.1;
    double targetRatio = (double) w / h;
    if (sizes == null) {
        return null;
    }
    Camera.Size optimalSize = null;
    double minDiff = Double.MAX_VALUE;
    int targetHeight = h;
    for (Camera.Size size : sizes) {
        double ratio = (double) size.width / size.height;
        if (Math.abs(ratio - targetRatio) > ASPECT_TOLERANCE)
            continue;
        if (Math.abs(size.height - targetHeight) < minDiff) {
            optimalSize = size;
            minDiff = Math.abs(size.height - targetHeight);
        }
    }
    if (optimalSize == null) {
        minDiff = Double.MAX_VALUE;
        for (Camera.Size size : sizes) {
            if (Math.abs(size.height - targetHeight) < minDiff) {
                optimalSize = size;
                minDiff = Math.abs(size.height - targetHeight);
            }
        }
    }
    return optimalSize;
}
```
在加载预览画面时，需要考虑Camera支持的尺寸（getSupportedPreviewSizes）和加载预览画面的SurfaceView的尺寸（layout_width/layout_height），在预览阶段，两者之间的关系直接影响清晰度及图像拉伸。对于Camera的尺寸，由于设备的硬件差异，不同设备支持的尺寸存在差异，但在默认情况（orientation=landscape）下，其width>height。以HTC609d为例，Camera支持的分辨率为1280*720（16：9）……640*480（4：3）……480*320（3：2）等十多种，而其屏幕的分辨率为960*540（16：9）。因此，很容易得到以下结论：（1）当Camera预览尺寸小于SurfaceView尺寸较多时，预览画面就不清晰；（2）Camera预览尺寸宽高比与SurfaceView宽高比相差较大时，预览画面就会拉伸。
上述代码在手机设置为横屏时并没有问题，在设置为竖屏时，为获得最优的预览尺寸，需要在调用此方法前比较SurfaceView的宽高。
```
if (mSupportedPreviewSizes != null) {
    mPreviewSize = getOptimalPreviewSize(mSupportedPreviewSizes, 
                        Math.max(width, height), Math.min(width, height));
}
```
获得与当前SurfaceView匹配的预览尺寸后，即可通过Camera.Parameters进行设置。
```
Camera.Parameters mParams = mCamera.getParameters();
mParams.setPreviewSize(mPreviewSize.width, mPreviewSize.height);
mCamera.setDisplayOrientation(90);

List<String> focusModes = mParams.getSupportedFocusModes();
if(focusModes.contains("continuous-video")){
    mParams.setFocusMode(Camera.Parameters.FOCUS_MODE_CONTINUOUS_VIDEO);
}
mCamera.setParameters(mParams);
```
## 三、视频清晰度及文件大小 ##
在第一节中讲到startRecordVideo，包括配置MediaRecorder，准备MediaRecorder以及启动，其中配置MediaRecorder是视频录制的重点，需要了解每项配置参数的作用，根据业务场景灵活配置。这里参考Google官方的示例给出一个可行的配置方案，然后再对其进行解释。
```
private void configureMediaRecorder() {
    // BEGIN_INCLUDE (configure_media_recorder)
    mMediaRecorder = new MediaRecorder();

    // Step 1: Unlock and set camera to MediaRecorder
    mCamera.unlock();
    mMediaRecorder.setCamera(mCamera);
    mMediaRecorder.setOrientationHint(90);

    // Step 2: Set sources
    mMediaRecorder.setAudioSource(MediaRecorder.AudioSource.VOICE_RECOGNITION);
    mMediaRecorder.setVideoSource(MediaRecorder.VideoSource.CAMERA);

    // Step 3: Set a Camera Parameters
    mMediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4);
    /* Fixed video Size: 640 * 480*/
    mMediaRecorder.setVideoSize(640, 480);
    /* Encoding bit rate: 1 * 1024 * 1024*/
    mMediaRecorder.setVideoEncodingBitRate(1 * 1024 * 1024);
    mMediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.H264);
    mMediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AAC);

    // Step 4: Set output file
    mMediaRecorder.setMaxFileSize(maxFileSizeInBytes);
    mMediaRecorder.setOutputFile(videoFilePath);
    // END_INCLUDE (configure_media_recorder)

    // Set MediaRecorder ErrorListener
    mMediaRecorder.setOnErrorListener(this);
}
```
**Step 1：**
setCamera参数能够使得在预览和录制中快速切换，避免Camera对象的重新加载。在某些Android手机自带的照相机程序中，切换预览与录制中的短暂卡顿，读者可自行体会。
mMediaRecorder.setOrientationHint(90)在录制方向为竖直（portrait）时使用，它能使视频文件的沿顺时针方向旋转90度，如果不设置此项，播放视频时，画面会发生90度的旋转。不过这里更重要的是，即使设置了此项，在某些播放器上，画面依然会有90度的旋转（比如将在手机上正常播放的视频导入到PC中进行播放，或者嵌入H5的video标签中），这可是为什么呢？注意setOrientationHint的说明：Note that some video players may choose to ignore the compostion matrix in a video during playback. 那么如何做到在所有播放器上都能以正常方向播放呢？稍等，后续专门对其进行说明。
**Step 2：**
setAudioSource(MediaRecorder.AudioSource.VOICE_RECOGNITION)，VOICE_RECOGNITION相比于MIC会根据语音识别的需要做一些调谐，当然，这需要在系统支持的情况下。
setVideoSource自然是VideoSource.CAMERA，只是在此两项设置必须在设置编码器之前设置，这无需说明。
**Step 3：**
setOutputFormat需要在Step 2之后，并且在prepare()之前。这里采用OutputFormat.MPEG_4格式。
setVideoSize需要权衡的因素较多，主要包括三方面：MediaRecorder支持的录制尺寸、视频文件的大小以及兼容不同Android机型。这里采用640 * 480（微信小视频的尺寸是320*240），文件大小在500-1000kb之间，并且市面上99%以上机型支持此录制尺寸。
setVideoEncodingBitRate与视频的清晰度有关，设置此参数需要权衡清晰度与文件大小的关系。太高，文件大不易传输；太低，文件清晰度低，识别率低。需要根据实际业务场景灵活调整。
setVideoEncoder采用H264编码，MPEG4、H263、H264等不同编码的差别比较可参考http://blog.csdn.net/wcl0715/article/details/676137，实际使用中，H264的压缩率较高，推荐使用。
setAudioEncoder采用AudioEncoder.AAC，该设置主要是考虑其通用性、兼容性。
**Step 4：**
setMaxFileSize指定录制文件的大小限制，当然还可以限制其最大录制时间。
setOutputFile指定输出视频的路径。
setOnErrorListener指定错误监听器。

在完成上述配置之后，即可准备MediaRecorder，并在返回成功后开始视频录制。

```
private boolean prepareConfiguredMediaRecorder() {
    // Step 5: Prepare configured MediaRecorder
    try {
        mMediaRecorder.prepare();
    } catch (Exception e) {
        releaseMediaRecorder();
        return false;
    }
    return true;
}
```
## 四、视频文件旋转 ##
第三节中Step 1提到对视频文件的旋转，因为某些播放器会忽略录制视频时的配置参数，因此可尝试通过第三方库对视频文件进行旋转，例如：OpenCV，fastCV等，在Camera对象的Camera.PreviewCallback中截取每帧数据byte[] data，然后对其进行处理，然后输出。该方法需要考虑处理方法的高效性，在编程时一般采用NDK，在C++中完成关键的处理，这里贴出fastCV中该处理方法的逻辑。
```
public void onPreviewFrame( byte[] data, Camera c ) {
    // Increment FPS counter for camera.
    util.cameraFrameTick();
    
    // Perform processing on the camera preview data.
    update( data, mDesiredWidth, mDesiredHeight );
    
    // Simple IIR filter on time.
    mProcessTime = util.getFastCVProcessTime();
    
    if( c != null ) {
        // with buffer requires addbuffer each callback frame.
        c.addCallbackBuffer( mPreviewBuffer );
        c.setPreviewCallbackWithBuffer( this );
    }
    
    // Mark dirty for render.
    requestRender();
}
```
其中，update为native方法，其实现由jni中对应的文件完成，其中调用了libfastcv.a中相应的API。这里涉及NDK编程的基本方法步骤：（1）开发环境；（2）编写Java代码、C/C++代码；（3）编译C/C++文件生成.so库；（4）重新编译工程，生成apk。由于本章不重点讲述NDK，这里不再展开。
除上述方法以外，笔者采用了另外一种思路进行了探索，上述方法处理的数据为每帧图像数据，可以理解为在线处理，而如果在录制完成之后再处理，可以理解为离线处理。这里采用了第三方库mp4parser，mp4parser是一款支持在Android中进行视频分割的库，这里通过其进行视频旋转。至于具体效果如何，读者有兴趣可自行尝试，这里留个悬念。
```
private boolean rotateVideoFileWithClockwiseDegree(String sourceFilePath, int degree) {
    if(!isFileAndDegreeValid(sourceFilePath, degree)) {
        return false;
    }
    rotateVideoFile(sourceFilePath, degree);
    return true;
}
```
对输入参数进行合法性检测之后，根据检测结果判断是否进行旋转。
```
private boolean isFileAndDegreeValid(String sourceFilePath, int degree) {
    if(sourceFilePath == null || (!sourceFilePath.endsWith(".mp4")) 
                              || (!new File(sourceFilePath).exists())) {
        return false;
    }
    if (degree == 0 || (degree % 90 != 0)) {
        return false;
    }
    return true;
}
```
```
private void rotateVideoFile(String sourceFilePath, int degree) {
    List<TrackBox> trackBoxes = getTrackBoxesOfVideoFileByPath(sourceFilePath);
    Movie rotatedMovie = getRotatedMovieOfTrackBox(trackBoxes);
    writeMovieToModifiedFile(rotatedMovie);
}
```
通过mp4parser旋转视频主要分为三步：（1）获取视频文件对应的TrackBoxes；（2）根据TrackBoxes获取旋转后的Movie对象；（3）将Movie对象写入文件。
```
private List<TrackBox> getTrackBoxesOfVideoFileByPath(String sourceFilePath) {
    IsoFile isoFile = null;
    List<TrackBox> trackBoxes = null;
    try {
        isoFile = new IsoFile(sourceFilePath);
        trackBoxes = isoFile.getMovieBox().getBoxes(TrackBox.class);
        isoFile.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
    return trackBoxes;
}
```
```
private Movie getRotatedMovieOfTrackBox(List<TrackBox> trackBoxes) {
    Movie rotatedMovie = new Movie();
    // 旋转
    for (TrackBox trackBox : trackBoxes) {
        trackBox.getTrackHeaderBox().setMatrix(Matrix.ROTATE_90);
        rotatedMovie.addTrack(new Mp4TrackImpl(trackBox));
    }
    return rotatedMovie;
}
```
```
private void writeMovieToModifiedFile(Movie movie) {
    Container container = new DefaultMp4Builder().build(movie);
    File modifiedVideoFile = new File(videoFilePath.replace(".mp4", "_MOD.mp4"));
    FileOutputStream fos;
    try {
        fos = new FileOutputStream(modifiedVideoFile);
        WritableByteChannel bb = Channels.newChannel(fos);
        container.writeContainer(bb);
        // 关闭文件流
        fos.close();
    } catch (Exception e) {
        e.printStackTrace();
    } 
}
```
本文对Android视频录制中常见的问题进行了说明，转载请注明出处（虽然也没什么转载）。