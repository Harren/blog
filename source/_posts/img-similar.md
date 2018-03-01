---
title: 相似图片搜索
date: 2017-07-19 19:45:18
categories:
- 图像处理
tags:
- 技术杂谈
---
该问题的提出主要是微博投出去的Banner广告被用户截图投诉，但是客服根据截图无法定位到具体的广告导致无法及时对广告进行下线操作，因此，需要开发一个对广告素材库进行以图搜图的这个需求。
最后采用dhash算法来实现这种图片相似搜索的功能。


## 平均哈希算法（aHash）
此算法是基于比较灰度图每个像素与平均值来实现的，最适用于缩略图，放大图搜索,其具体的步骤如下：
1. **图片缩放**。为了保留结构去掉细节，去除大小、横纵比的差异，把图片统一缩放到8*8，共64个像素的图片。
2. **灰度图转换**。把缩放后都图片转化为256阶的灰度图。
3. **平均值计算**。计算进行灰度处理后图片的所有像素点的平均值。
4. **像素值比较**。遍历灰度图的像素，如果改像素点点值大于平均值，则计为1，否则，计为0。
5. **获取指纹**。将8*8的图片按照固定的顺序展开成64为的bit串，得到该图片的指纹。组合的次序并不重要，只要保证所有图片都采用同样次序就行了（例如，自左到右、自顶向下、big-endian）。
6. **汉明距离计算**。对比两张图片的指纹，计算汉明距离(从一个字符串到另一个字符串需要变换几次)，汉明距离越小则说明图片越相似，当距离为0时，则认为两张图完全相同，反之，则汉明距离越大越不一致。

## 感知哈希算法（pHash）
平均哈希算法不够精确，更适合搜索缩略图，为了获得梗精确的结果可以选择感知哈希算法，感知哈希算法利用DCT(离散余弦变换)来降低频率，其算法实现如下:
1. **图片缩放**。为了保留结构去掉细节，去除大小、横纵比的差异，同时考虑到DCT的计算，把图片统一缩放到32*32。
2. **灰度图转换**。把缩放后都图片转化为256阶的灰度图。
3. **计算DCT**。DCT是把图片分解频率聚集和梯状形。
4. **缩小DCT**。获取到左上角的8*8的矩阵，这部分呈现来图片中的低频信息。
5. **计算平均值**。计算所有64个值的平均值。
6. **进一步减小DCT**。这是最主要的一步，根据8*8的DCT矩阵，设置0或1的64位的hash值，大于等于DCT均值的设为”1”，小于DCT均值的设为“0”。结果并不能告诉我们真实性的低频率，只能粗略地告诉我们相对于平均值频率的相对比例。只要图片的整体结构保持不变，hash结果值就不变。能够避免伽马校正或颜色直方图被调整带来的影响。
7. **计算哈希值**。将64bit设置成64位的长整型，组合的次序并不重要，只要保证所有图片都采用同样次序就行了。将上一步的比较结果，组合在一起，就构成了一个64位的整数，这就是这张图片的指纹。
8. **汉明距离计算**。对比两张图片的指纹，计算汉明距离(从一个字符串到另一个字符串需要变换几次)，汉明距离越小则说明图片越相似，当距离为0时，则认为两张图完全相同，反之，则汉明距离越大越不一致。

## 差异哈希算法（dHash）
相比pHash，dHash的速度要快的多，相比aHash，dHash在效率几乎相同的情况下的效果要更好，它是基于渐变实现的。其实现步骤如下：
1. **图片缩放**。为了保留结构去掉细节，去除大小、横纵比的差异，把图片统一缩放到8*8，共64个像素的图片。
2. **灰度图转换**。把缩放后都图片转化为256阶的灰度图。 
3. **计算差异值**。dHash算法工作在相邻像素之间，这样每行9个像素之间产生了8个不同的差异，一共8行，则产生了64个差异值。
4. **获得指纹**。大于0的差异值计为1，小于0的差异值计为0。
5. **计算哈希值**。将64bit设置成64位的长整型，组合的次序并不重要，只要保证所有图片都采用同样次序就行了。将上一步的比较结果，组合在一起，就构成了一个64位的整数，这就是这张图片的指纹。
6. **汉明距离计算**。对比两张图片的指纹，计算汉明距离(从一个字符串到另一个字符串需要变换几次)，汉明距离越小则说明图片越相似，当距离为0时，则认为两张图完全相同，反之，则汉明距离越大越不一致。

dhash算法的实现代码如下所示：

```Java
public class Dhash {

    private static String ERROR_IMG = "0000000000000000000000000000000000000000000000000000000000000000";

    /**
     * 返回64位hash
     * @param filePath String
     * @return String
     */
    public static String dHash(String filePath) throws IOException {

        BufferedImage src = ImageIO.read(new File(filePath));
        if(src == null) {
            return null;
        }
        Image image = src.getScaledInstance(9, 8, Image.SCALE_DEFAULT);
        BufferedImage tag = new BufferedImage(9, 8, BufferedImage.TYPE_INT_RGB);
        Graphics g = tag.getGraphics();
        g.drawImage(image, 0, 0, null);
        g.dispose();

        //转灰度
        ColorSpace cs = ColorSpace.getInstance(ColorSpace.CS_GRAY);
        ColorConvertOp op = new ColorConvertOp(cs, null);
        tag = op.filter(tag, null);

        int minx = tag.getMinX();
        int miny = tag.getMinY();

        String bitStr = "";
        int[][] mask = new int[8][8];
        for(int i = minx;i < 8;i++) {
            for(int j = miny;j < 8;j++) {
                int pixel = tag.getRGB(i, j);
                int next_pixel = tag.getRGB(i+1, j);
                int gray = (pixel & 0xff0000) >> 16;
                int nextGray = (next_pixel & 0xff0000) >> 16;
                mask[i][j] = nextGray - gray;

                bitStr += (mask[i][j] < 0) ?  "1" : "0";

            }
        }
        return bitStr;
    }


    /**
     * 缩放图像
     * @param srcImageFile String
     * @param result String
     * @param height int
     * @param width  int
     */
    private static void scale(String srcImageFile, String result, int height, int width) {

        try {
            BufferedImage src =ImageIO.read(new File(srcImageFile)) ;
            Image image = src.getScaledInstance(width, height, Image.SCALE_DEFAULT);
            BufferedImage tag = new BufferedImage(width, height,
                    BufferedImage.TYPE_INT_RGB);
            ImageIO.write(tag, "JPEG", new File(result));// 输出到文件流

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 彩色转为黑白
     * @param srcImageFile 源图像地址
     * @return BufferedImage
     */
    private static BufferedImage gray(String srcImageFile) {
        try {
            BufferedImage src = ImageIO.read(new File(srcImageFile));
            ColorSpace cs = ColorSpace.getInstance(ColorSpace.CS_GRAY);
            ColorConvertOp op = new ColorConvertOp(cs, null);
            src = op.filter(src, null);
            return src;
            //ImageIO.write(src, "JPEG", new File(destImageFile));
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```


## 参考文献
1. [感知哈希算法（pHash）](http://www.phash.org/)