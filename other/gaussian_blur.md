[<< 返回到主页](index.md)  

**这里将介绍高斯模糊的博客文章**  

高斯模糊算法主要是将原图像变得模糊。主要是二元正态分布构成，然后获取一个二维大小的权重系数，作用在图像像素之上进行加权求和。  

HexGaussBlurer.h:  
```
class HexGaussBlurer
{
private:
    int gaussFi;
    float** gaussValue;
    unsigned __int32 *colorData;
    FastMath::Point4PS *gaussWeightedColor;
public:
    HexGaussBlurer()
    {
        gaussValue = 0;
        colorData = 0;
        gaussWeightedColor = 0;
    }

    ~HexGaussBlurer()
    {
        if (gaussValue)
        {
            for (int i=0; i<2*gaussFi+1; i++)
                if (gaussValue[i])
                    delete [] gaussValue[i];
        }
        delete [] gaussValue;
        if (colorData)
            delete [] colorData;
        if (gaussWeightedColor)
            _aligned_free(gaussWeightedColor);
    }

    void ApplyGaussBlur(unsigned __int32 *bitmap, unsigned __int32 width, unsigned __int32 height, float gaussRadius);

    void InitGauss(float gaussRadius);

    void GetWeightedColor(float gaussRadius, unsigned __int32 *bitmap, unsigned __int32 width, unsigned __int32 height);

    unsigned __int32 GetFilterColor(unsigned __int32 w, unsigned __int32 h, unsigned __int32 width, unsigned __int32 height);

};
```



HexGaussBlurer.cpp:  
```
void HexGaussBlurer::ApplyGaussBlur(unsigned __int32 *bitmap, unsigned __int32 width, unsigned __int32 height, float gaussRadius)
{
    GetWeightedColor(gaussRadius, bitmap, width, height);
    for (unsigned int h=0; h<height; ++h)
    {
        unsigned int rowIndex = h * width;
        for (unsigned int w=0; w<width; ++w)
        {
            unsigned __int32 newColor = GetFilterColor(w, h, width, height);
            newColor |= 0xff000000;
            bitmap[rowIndex + w] = newColor;
        }
    }
}

void HexGaussBlurer::InitGauss(float gaussRadius)
{
    static const float _c2PI = 2.0f * 3.1415927f;
    gaussFi = 3;
    gaussValue = new float*[2*gaussFi+1];
    for (int i=0; i<2*gaussFi+1; ++i)
    {
        gaussValue[i] = new float[2*gaussFi+1];
    }
    float sigma = gaussRadius;
    float powsigma = sigma * sigma;
    float A = 1.0f / (_c2PI * powsigma);
    for (int x=-gaussFi; x<=gaussFi; ++x)
    {
        for (int y=-gaussFi; y<=gaussFi; ++y)
        {
            float ex = 0;
            float valuebuf = -(x*x + y*y)/(2*(powsigma));
            if (valuebuf>=-88 && valuebuf<=88)
                FastMath::FastExp(valuebuf, &ex);
            else
                ex = exp(valuebuf);
            gaussValue[x+gaussFi][y+gaussFi] = A * ex;
        }
    }
}

void HexGaussBlurer::GetWeightedColor(float gaussRadius, unsigned __int32 *bitmap, unsigned __int32 width, unsigned __int32 height)
{
    InitGauss(gaussRadius);

    colorData = new unsigned __int32[width*height];
    memcpy(colorData, bitmap, sizeof(unsigned __int32)*width*height);

    if (gaussFi > 0)
    {
        int valueNum = (int)(((gaussFi+2)*(gaussFi+1)) * 0.5f);
        int sumn = valueNum * width * height;
        gaussWeightedColor = (FastMath::Point4PS *)_aligned_malloc(sizeof(FastMath::Point4PS) * sumn, 16);
        for (int i=0; i<sumn; ++i)
        {
            gaussWeightedColor[i] = _mm_set1_ps(0.0f);
        }

        for (unsigned int h=0; h<height; ++h)
        {
            unsigned int rowIndex = h * width;
            for (unsigned int w=0; w<width; ++w)
            {
                unsigned int cr = colorData[rowIndex + w]; 
                int vn = 0;
                for (int i=0; i<=gaussFi; ++i)
                {
                    for (int j=0; j<=i; ++j)
                    {
                        float weight = gaussValue[j+gaussFi][i+gaussFi];
                        FastMath::Point4PS curCr = {0.0f, 0.0f, 0.0f, 0.0f};
                        curCr.m128_f32[0] = (cr & 0xff) * weight;
                        curCr.m128_f32[1] = ((cr >> 8) & 0xff) * weight;
                        curCr.m128_f32[2] = ((cr >> 16) & 0xff) * weight;
                        curCr.m128_f32[3] = ((cr >> 24) & 0xff) * weight;
                        gaussWeightedColor[(rowIndex + w) * valueNum + vn] = curCr;
                        ++ vn;
                    }
                }
            }
        }
    }
}

unsigned __int32 HexGaussBlurer::GetFilterColor(unsigned __int32 w, unsigned __int32 h, unsigned __int32 width, unsigned __int32 height)
{
    FastMath::Point4PS rgb = {0.0f, 0.0f, 0.0f, 0.0f};
    int valueNum = ((gaussFi+2)*(gaussFi+1))/2;
    for (int y=-gaussFi; y<=gaussFi; ++y)
    {
        int curh = h+y;
        for (int x=-gaussFi; x<= gaussFi; ++x)
        {
            int curw = w+x;
            if (curw>=0 && curw<(int)width && curh>=0 && curh<(int)height)
            {
                int xx = abs(x);
                int yy = abs(y);
                int vn = 0;
                if (xx==yy)
                {
                    int trieverynum = 2;
                    int beforenum = 0;
                    for (int yyi=0; yyi<yy; ++yyi)
                    {
                        beforenum += trieverynum;
                        ++trieverynum;
                    }
                    vn = beforenum;
                }
                else if(yy>xx)
                {
                    int trieverynum = 1;
                    int beforenum = 0;
                    for (int yyi=0; yyi<yy; ++yyi)
                    {
                        beforenum += trieverynum;
                        ++trieverynum;
                    }
                    beforenum += xx;
                    vn = beforenum;
                }
                else if (yy<xx)
                {
                    int trieverynum = 1;
                    int beforenum = 0;
                    for (int xxi=0; xxi<xx; ++xxi)
                    {
                        beforenum += trieverynum;
                        ++trieverynum;
                    }
                    beforenum += yy;
                    vn = beforenum;
                }
                FastMath::Point4PS cr = gaussWeightedColor[(curh * width + curw) * valueNum + vn];
                rgb = _mm_add_ps(rgb, cr);
            }
        }
    }
    const FastMath::Point4PS minC = {0.0f, 0.0f, 0.0f, 0.0f}; 
    const FastMath::Point4PS maxC = {255.0f, 255.0f, 255.0f, 255.0f};
    rgb = _mm_min_ps(rgb, maxC);
    rgb = _mm_max_ps(rgb, minC);
    unsigned char cr = unsigned char(rgb.m128_f32[0]);
    unsigned char cg = unsigned char(rgb.m128_f32[1]);
    unsigned char cb = unsigned char(rgb.m128_f32[2]);
    unsigned char ca = (colorData[h * width + w] >>24) & 0xff;
    unsigned __int32 nowcolor = cr | (cg << 8) | (cb << 16) | (ca << 24);
    return nowcolor;
}
```