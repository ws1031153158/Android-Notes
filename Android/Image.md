# Bitmap
## Load
decodeFile/decodeResource 将 drawble 转化为 bitmap，decodeStream.decodeByteArray 通过 imageView 显示 bitmap  
需要通过裁剪来提高加载效率，降低内存占用，一般通过 BitmapFactory.options 实现，inJustDecodeBounds = true 表示只解析传入的宽高而不加载。  
关键是指定合适的采样率，inSampleSize（大于 1 的整数），一般 scale 为 采样率平方的倒数
# Drawable
是一个抽象类，具体实现在各个子类，不一定有宽高，如只有颜色的 drawable，且内部宽高不等于本身大小，一般没有大小这个概念，只是填充 view 是会拉伸而已
# Rect & RectF
都是创建矩形，都实现 Parcelabel 接口。   
Rect 是 final 类，参数是 int ， RectF 是普通类，参数是 float，RectF 精度高，此外，有些方法不一致。
