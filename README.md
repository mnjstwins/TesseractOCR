TesseractOCR
============

For information on the Noisebridge email group and scanner please go to https://noisebridge.net/wiki/Digital_Archivists

This repo documents full text extraction from archival photos of books using an older model of the Daniel Reetz book scanner (https://www.noisebridge.net/wiki/Bookscanner). We are currently using custom software for the digital capture of images via usb (https://github.com/danyq/diybookscanner). Post processing is done locally or using cloud services with the open source software Tesseract.

![Schematic of the Reetz book scanner](https://github.com/andrewdefries/TesseractOCR/blob/master/Reetz_Scanner_Schematic.png)

Installing Tesseract on linux machines
======================================

Tesseract is a freely available OCR software that can perform command line text extraction from images. Tesseract also has langauge support for multiple languages. 

See the Tesseract page for more info  https://code.google.com/p/tesseract-ocr/ 

You require a number of modules to get this workflow moving.

To build Tesseract working environment do (tested in ubuntu and debian)
```
# tesseract dependencies
sudo apt-get install autoconf automake libtool
sudo apt-get install libpng12-dev
sudo apt-get install libjpeg62-dev
sudo apt-get install libtiff4-dev
sudo apt-get install zlib1g-dev
sudo apt-get install libicu-dev # (if you plan to make the training tools)
sudo apt-get install libleptonica-dev
sudo apt-get install tesseract-ocr

# otherwise install from source
wget http://tesseract-ocr.googlecode.com/files/tesseract-ocr-3.01.eng.tar.gz
tar xf tesseract-ocr-3.01.eng.tar.gz

# imagemagick is a popular command line tool to perform image manipulations
sudo apt-get install imagemagick
sudo apt-get install graphicsmagick-libmagick-dev-compat
sudo apt-get install exactimage

# also copy to local the textcleaner.sh script
wget http://www.fmwconcepts.com/imagemagick/downloadcounter.php?scriptname=textcleaner&dirname=textcleaner

# consider getting potrace to make svg traces of images
 
```

Post Processing Workflow
========================

![Workflow of the Reetz book scanner](https://github.com/andrewdefries/TesseractOCR/blob/master/ReetzWorkFlow.png)

Briefly, the workflow is as follows to extract text from high quality images. Take pictures of source material, pre-process using imagemagick scripts (rotate, contrast, crop), input image to tesseract. 

In our capture setup we used two digital cameras facing both the even and odd page for simultaneous page capture. This resulted in two images that required a rotation 90 degrees to the original page position. Imagemagick was used to rotate raw images.

```
# for odd pages
convert -rotate +90 input_image.jpg output_image.jpg

# for even pages
convert -rotate -90 input_image.jpg output_image.jpg
```

A number of options are available to pre-process and determine the appropriate crop area. Here we use a combination of both gui and command line tools. The open source image editing tool FIJI or Fiji is Just Image J ( http://fiji.sc/wiki/index.php/Fiji) is available for many platforms and can be scripted into custom workflow in several ways. Fiji was used to determine the appropriate crop area which was saved to a file. We send the user a sample of images to set the crop box area using FIJI.

GetSampleForUser.sh
```
targetbucket=(`cat TargetBucket`)

gsutil -m ls $targetbucket | shuf > ShuffledWorkList
######
cat ShuffledWorkList | sed -n '/.*[02468]\.jpg/p' | head -n 18 > EvenCropWorkList
cat ShuffledWorkList | sed -n '/.*[13579]\.jpg/p' |head -n 18 > OddCropWorkList
######
download4usereven=(`cat EvenCropWorkList`) 
download4userodd=(`cat OddCropWorkList`) 
######
# the download the files to local 
```

To crop the page to a specified area we supply imagemagick with command line options for the x and y bounding box to crop
```
cropeven=(`cat CropValue`)

convert input_image.jpg -crop $cropvalue outut_image.jpg
```

Tesseract takes the following command line options to convert an image to text
```
############
remainder=(`cat WorkList`)
############
for i in "${remainder[@]}"
do
######
targetbucket=(`cat TargetBucket`)
gsutil -m cp $targetbucket/$i.jpg .
echo $i >> RunLog
convert $i.jpg -crop $cropeven $i.crop.jpg 
convert -rotate -90 $i.crop.jpg $i.rotated.jpg
rm $i.crop.jpg
./textcleaner.sh -g -e stretch -f 25 -o 5 -s 1 $i.rotated.jpg $i.ready.jpg
rm $i.rotated.jpg
######
convert $i.ready.jpg $i.ready.pnm
potrace $i.ready.pnm -s
convert $i.ready.svg $i.ready.jpg
rm $i.ready.pnm
######
tesseract $i.ready.jpg $i
tesseract $i.ready.jpg $i hocr
hocr2pdf -i $i.ready.jpg -o $i.pdf < $i.html
######
```

Software Updates
=======
The folder CustomCrop contains some scripts to multi-thread the OCR session, now for 4 cores. Have to replace some values with system variables such as working directory and number of cores.


