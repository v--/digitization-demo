# Digitization of a scanned paper

I have found a scanned paper of von Neumann [here](https://cs.uwaterloo.ca/~y328yu/classics/vonNeumann.pdf). It is already a compiled PDF file, but it is not digitized in the usual sense --- it is simply a sequence of colored images. There is no image "unpapering", no metadata, no text layer. This makes it a good candidate for demonstrating my digitization workflow. Roughly, we transform into [this file](<./vonNeumann.pdf>) into [this file](<./John von Neumann - On the Theory of Games of Strategy.pdf>).

## Workflow

Scanning is more-or-less obvious. The processing work is done in a series of steps, each handled by different tools.

The orchestrator for the work is [GNU Make](https://www.gnu.org/software/make/). Roughly, I can run `make -j 8` and leave 8 workers to do all the heavy work concurrently. The [Makefile](./Makefile) has detailed comments on every aspect of the workflow.

The name of the working directory is taken into account, in particular
* For the path for the "intermediate" file storage (`/var/tmp/digitization/1e352581`).
* For the output file name ([`John von Neumann - On the Theory of Games of Strategy.pdf`](<./John von Neumann - On the Theory of Games of Strategy.pdf>)).

Other auxiliary programs are of course necessary. The ones listed below have been most beneficial in general:

* [ImageMagick](https://imagemagick.org/) for general-purpose image processing.
* [unpaper](https://github.com/unpaper/unpaper) for post-processing scanned pages.
* [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) for OCR.
* [OCRmyPDF](https://github.com/ocrmypdf/OCRmyPDF) for performing OCR on an existing PDF file via the above tool.
* [Ghostscript](https://www.ghostscript.com/) for processing PDF files.
* [djvulibre](https://github.com/traycold/djvulibre) for working with DjVu files.
* [dpsprep](https://github.com/kcroker/dpsprep) for converting DjVu to PDF.
* [pdftk](https://www.pdflabs.com/tools/pdftk-the-pdf-toolkit/) for some common PDF operations.

### Starting point

![The raw image of page 39](./demo_images/0027_scanned.png)

This is simply an image as taken from the scanner. In this case, it is only for demonstration since we use `pdftoppm` to extract a grayscale image directly.

### Image processing

![The processed image of page 39](./demo_images/0027_magick.png)

This is the result of passing the above image through several ImageMagick operations. The concrete operations are commented in the Makefile in detail. The gist is that we take a colorful or grayscale image and do our best to produce a monochrome image out of it.

### Unpapering

![The unpapered image of page 39](./demo_images/0027_unpaper.png)

An immensely useful operation is running the above image through the `unpaper` program, which cleans up various visual artifacts and does some page alignment.

### Combining the images

Once all pages are processed individually, we combine them into a single large PDF file using ImageMagick. At this step we also adjust the DPI that the resulting PDF file must have. There are no visual changes from this point on.

We avoid compression here because it will be done later by OCRmyPDF, but it is important that we have done our best to produce images that _can_ be compressed efficiently using [group4](https://en.wikipedia.org/wiki/Group_4_compression) or [jbig2](https://en.wikipedia.org/wiki/JBIG2). Non-monochrome PDF image files cannot be compressed efficiently, leading to files that are hundreds of megabytes large.

> [!NOTE]
> [DjVu](http://yann.lecun.com/ex/djvu/index.html) does miracles for both text layers and compression, making it more convenient than PDF. It is unfortunately more-or-less a dead file format at this point and tools for working with it are in maintenance mode at best.
> I used to build DjVu files, but I learned that it's better to provide a PDF myself than to let people use bad converters.

### OCR

We have a PDF file with nice readable text, but the text only exists as an image. It cannot be searched nor copied.

OCRmyPDF helps passing the images through OCR software like Tesseract. More importantly, it does a great job at compressing the source files --- I've had it compress the files by 92% (although, as mentioned above, the images were preprocessed as to allow such compression).

### Outline

The final step of the build process is the so-called outline --- mostly bookmarks and page index translation. The bookmarks are obvious. Page translation less so, but it is born out of necessity; for example, the 27th page in the PDF file is numbered "39", and we want the PDF metadata to reflect that.

We use pdftk for adding this metadata. See the [./outline.txt](./outline.txt) file for the format used.
