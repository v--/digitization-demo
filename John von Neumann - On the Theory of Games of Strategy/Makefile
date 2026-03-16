# This is a GNU Make file specifically designed for building PDF books from a directory of images.
# It can process multiple images concurrently with `make --jobs=8`.
#
# Every book is different and has different processing requirements,
# and it often makes sense copying this file and adapting it for an individual book.
# Adapting usually involves modifying in non-trivial ways the targets for the individual pages.
#
# I do not like to put licenses on my code, so consider this Makefile Unlicensed (https://unlicense.org/).

# Configuration variables
OCR_LANGUAGES := eng+grc # Math books and papers generally feature Latin and Greek letters (even if they are written in e.g. Cyrillic)
PAGE_RANGE := $(shell seq --format '%04g' 1 30)
EXECUTABLES := pdftoppm ddjvu magick unpaper ocrmypdf pdftk # The list of binaries whose non-existence will fail the build
SOURCE_PDF := vonNeumann.pdf

# Some technical variables
E := # This whitespace trick is from https://stackoverflow.com/a/56411000
OUTPUT_NAME_RAW := $(notdir $(shell pwd))
OUTPUT_NAME := $(subst $E $E,\ ,$(OUTPUT_NAME_RAW))
CHECKSUM := $(shell echo $(OUTPUT_NAME) | b2sum --length 32 --zero | tr --delete ' -')
TMP_DIR := /var/tmp/digitization/$(CHECKSUM)
PROCESSED_IMAGES := $(addprefix $(TMP_DIR)/,$(addsuffix _unpaper.pbm,$(PAGE_RANGE)))

# Check if the necessary executables exist
MISSING_DEPS := $(strip $(foreach exec, $(EXECUTABLES),\
	$(if $(shell which $(exec) 2>/dev/null),,$(exec)) \
))

$(if $(MISSING_DEPS),$(error Missing executables: $(MISSING_DEPS)),)

# If the images are given as a directory of files, the following is helpful in ensuring that there are no "missing" pages:
# MISSING_PAGES := $(strip $(foreach page, $(PAGE_RANGE),\
# 	$(if $(wildcard $(page).*),,$(page)) \
# ))
#
# $(if $(MISSING_PAGES),$(error Missing pages: $(MISSING_PAGES)),)

# Targets that not explicitly mentioned get implicitly marked as intermediate, and may get deleted.
# If the end result is not satisfying, we will need to regenerate all images again rather than only modify some of them.
# .NOTINTERMEDIATE with no prerequisites causes all targets to be treated as not intermediate.
# Once we are done, we can use `make clean-tmp` to remove the intermediate files.
.NOTINTERMEDIATE:
.PHONY: clean-tmp clean-pdf

# The main target that gets made by default
$(OUTPUT_NAME).pdf: $(TMP_DIR)/ocr.pdf outline.txt
	pdftk $< update_info_utf8 outline.txt output '$@'

clean-tmp:
	rm --recursive --force $(TMP_DIR)

clean-pdf:
	rm --force '$(OUTPUT_NAME_RAW).pdf'

$(TMP_DIR):
	mkdir --parents $(TMP_DIR)

# We usually have a directory of source images, but sometimes we must re-process an existing PDF or DjVu file.
# In this case, we take the source file and extract images from it.
# These will get saved as 0001_pre.pgm, 0002_pre.pgm in the temporary directory.
$(TMP_DIR)/%_pre.pgm: $(SOURCE_PDF) | $(TMP_DIR)
	pdftoppm -f $* -l $* -singlefile -gray $< $(TMP_DIR)/$*_pre

# ImageMagick is massively useful in general.
#
#   * "-resize ..." increases the size of the image so that the convention to monochrome produces better results.
#     Here we specify the smaller dimension (width) and the larger dimension (height) scales according to the aspect ratio.
#
#   * "-gravity ... -crop ..." crops the width. The "gravity" pulls the odd pages to the bottom left and the even pages to the top right.
#
#   * "-auto-level" is generally useful to scale the levels of gray before quantization.
#
#   * "-brightness-contrast ..." is also often beneficial to manually adjust brightness and contrast
#
# Because of its .pbm extension, the result will be a monochrome image.
# By default imagemagick uses unsophisticated thresholding, which works well for high-quality images at high resolution.
# For finer control over quantization, especially for lower-quality images, we can use local thresholding (e.g. -lat 50x50-20%).
# If the lighting is uniform, ordered dithering (-ordered-dither o2x2) can also be helpful.
$(TMP_DIR)/%_magick.pbm: $(TMP_DIR)/%_pre.pgm
	magick $< \
		-resize '2500' \
		-gravity $(if $(filter $(shell expr $* % 2),1),SouthEast,NorthWest) -crop '2100x2970+0+0' +repage \
		-auto-level \
		-brightness-contrast -25x50% \
		$@;

# Unpaper does wonders in page preparation
$(TMP_DIR)/%_unpaper.pbm: $(TMP_DIR)/%_magick.pbm
	unpaper $< $@

# It is often helpful to specialize
# For example, the first page has a large padding from the top.
# unpaper tries to center the content of the page, which in this case is undesirable.
# So --no-border-align makes sure the top padding is preserved.
$(TMP_DIR)/0001_unpaper.pbm: $(TMP_DIR)/0001_magick.pbm
	unpaper --no-border-align $< $@

$(TMP_DIR)/combined.pdf: $(PROCESSED_IMAGES)
	magick $^ -define pdf:Title='$(OUTPUT_NAME_RAW)' -density 300 -verbose $@

$(TMP_DIR)/ocr.pdf: $(TMP_DIR)/combined.pdf
	ocrmypdf --optimize 3 --language=$(OCR_LANGUAGES) $< $@
