---
title: Portfolio |  Basic Image Processing
---

[Home](../../../index.md) / [Software Projects](index.md) /

# Basic Image Processing
The basic image processing application is a command line interface tool to apply various effects to an image. The program was developed in C#, using bitmaps to load the image. However, the applied algorithm can be used in any programming language. These effects include:
- saturation
- blur
- colour inversion
- pixelation
- normalisation
- normal mapping

## Reading PNG data
the png file format is decoded in programming languages as a singular large byte array structure, indexed similar to a flattened 2D array. The array formats the data allocating 1 byte per channel (RGBA), representing 0 to 255. So, the array looks like: RGBARGBARGBA...  The main basis for the following algorithms is to loop over the array, extract the RGBA values, apply filters to these values, combine it back into a single double, and write to an output file. The formula to index the start of each pixel is: (y * stride) + (x * bytesPerPixel). The stride is the amount of bytes needed to move down a row, which is required for searching pixels directly above or below the target.

The main code to extract the pixel data is given in the following function, where the variables are stored in global variables to be retrieved later from getters:

```C#
public ImageLoader(string filePath){
    try{
        imageMap = new Bitmap(filePath);

        Rectangle rect = new Rectangle(0, 0, imageMap.Width, imageMap.Height);

        BitmapData imageData = imageMap.LockBits(rect, ImageLockMode.ReadWrite, imageMap.PixelFormat);

        IntPtr pixelPointer = imageData.Scan0;
        int bytes = Math.Abs(imageData.Stride) * imageMap.Height;
        pixels = new byte[bytes];

        imageStride = imageData.Stride;

        System.Runtime.InteropServices.Marshal.Copy(pixelPointer, pixels, 0, bytes);

        imageMap.UnlockBits(imageData);
    }
    catch{
        throw new Exception("Image failed to load");
    }
}
```

## Saturation
There are many ways to modify the saturation of a colour pixel, but the method I used was to convert from RGB to HSL format, as saturation was one of the parameters. So, changing the saturation is as simple as converting to HSL, adjusting the S value, and converting back to RGB. The function to convert to HSE is given:
```C#
public static (float, float, float) RGB_To_HSL(float red, float green, float blue){
    blue  /= 255;
    green /= 255;
    red   /= 255;

    //convert rgb colour to hsv values
    float maxColour = Math.Max(blue, Math.Max(green,red));
    float minColour = Math.Min(blue, Math.Min(green,red));
    float luminance = (maxColour + minColour) / 2;

    float delta = maxColour - minColour;

    float saturation = 0;
    float hue = 0;

    if(maxColour != minColour){
        saturation = luminance <= 0.5f ? delta/(maxColour+minColour) : delta/(2-maxColour-minColour);

        if(maxColour == red){
            hue = (green - blue) / delta;
        }
        else if(maxColour == green){
            hue = 2 + (blue - red) / delta;
        }
        else if(maxColour == blue){
            hue = 4 + (red - green) / delta;
        }
        hue *= 60;
        if(hue < 0){
            hue += 360;
        }
    }

    return (hue, saturation, luminance);
}
```

## Blur
Blur is achieved by setting the colour value of a pixel to the average of its neighbors. Since the pixel values are no longer independent, the average colour values for each pixel must be written to a temporary array instead of asjusting the values in place. The code to compute the average for a cell is given:
```C#
 private byte[] GetAverageColourByBlock(byte[] pixels, int xCoord, int yCoord, int imageStride, int bytesPerPixel,int imageHeight, int imageWidth){
    int newWidth = (int)Math.Floor((double)matrixWidth/2); //half the value, so it is added and subracted from the middle pixel (only works properly if odd, since the lost 0.5 is the starting pixel);
    int minX = (xCoord - newWidth >= 0) ? xCoord-newWidth : 0;
    int maxX = (xCoord + newWidth < imageWidth) ? xCoord+newWidth : imageWidth-1;
    int minY = (yCoord - newWidth >= 0) ? yCoord-newWidth : 0;
    int maxY = (yCoord + newWidth < imageHeight) ? yCoord+newWidth : imageHeight-1;

    float[] colours = new float[bytesPerPixel];

    //loop through each pixel within the xy bounds, to obtain their colour value for the average
    int numOfPixels = 0;
    for(int y = minY; y <= maxY; y++){
        for(int x = minX; x <= maxX; x++){
            numOfPixels ++;
            int index = (y*imageStride) + (x*bytesPerPixel);
            for(int i = 0; i < colours.Length; i++){
                colours[i] = colours[i] + pixels[index + i];
            }
        }
    }

    //calculate the average pixel colour across the region and apply it to a new return array
    byte[] cols = new byte[colours.Length];
    for(int i = 0; i < colours.Length; i++){
        colours[i] = (byte)(colours[i] / numOfPixels);
        cols[i] = (byte)Math.Min(colours[i], 255);
    }
    return cols;
}

```

## Inverter
Inverts the colours by simply setting: data = 255 - data per pixel. This is the simplest algorithm, so there is not much to explain.

## Pixelising.
Pixelising an image is done the same way as bluring, by taking the average of neighbors. However, instead of setting the pixel as the center of the averaging window, the windows are predefined into blocks, representing the resolution of the pixelised image. The average is computed per channel, to give a more accurate colour as apposed to averaging all bytes.


