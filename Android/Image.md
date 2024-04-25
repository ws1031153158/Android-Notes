# Bitmap
## Load
decodeFile/decodeResource 将 drawble 转化为 bitmap，decodeStream.decodeByteArray 通过 imageView 显示 bitmap  
需要通过裁剪来提高加载效率，降低内存占用，一般通过 BitmapFactory.options 实现，inJustDecodeBounds = true 表示只解析传入的宽高而不加载。  
关键是指定合适的采样率，inSampleSize（大于 1 的整数），一般 scale 为 采样率平方的倒数
# Drawable
是一个抽象类，具体实现在各个子类，不一定有宽高，如只有颜色的 drawable，且内部宽高不等于本身大小，一般没有大小这个概念，只是填充 view 是会拉伸而已
## IntrinsicWidth/Height
可以用来获取 ImageView 中显示图片的大小，但是在某些情况下，返回的大小可能与原图的大小不一致，这个值的大小与运行设备的当前 dpi 有关，当该图片所在的图片文件夹对应的分辨率与设备运行的分辨率一致时，拿到的宽度才是原始图片的宽度。  
## Tips
getWidth 和 getHeight 拿到的是 ImageView 的宽高，同样会受到设备分辨率的影响，比如给 ImageView 设置为 100dip*100dip，在像素密度为 3 的手机上，getWidth 和 getHeight 拿到的结果就是 300px*300px
# Rect & RectF
都是创建矩形，都实现 Parcelabel 接口。   
Rect 是 final 类，参数是 int ， RectF 是普通类，参数是 float，RectF 精度高，此外，有些方法不一致。
# Tips
图片占用的内存大小公式：分辨率 * 每个像素点的大小  
某些场景下，位于 res 内的不同资源目录中的图片，被加载进内存时的分辨率会经过一层转换，此时的分辨率已不是图片本身的分辨率了  
转换后的分辨率：新图 = 原图 * (设备的 dpi / 目录对应的 dpi )     
系统默认为 ARGB_8888 作为像素点的数据格式  
ALPHA_8 -- (1B)  
RGB_565 -- (2B)  
ARGB_4444 -- (2B)  
ARGB_8888 -- (4B)  
RGBA_F16 -- (8B)  
