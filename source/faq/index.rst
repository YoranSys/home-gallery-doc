FAQ
===

What data is requested from public services?
--------------------------------------------

The goal of HomeGallery is to use as less public serivces as possible
due sensitive private image data. It tries to use service which can
be deployed local. However the setup requires technical knowlege and
technical maintenance. Following services are called:

For geo reverse lookups (resolve geo coordinates to addess), HomeGallery
queries the `Nominatim Service <https://nominatim.openstreetmap.org/reverse>`_
from `OpenStreetMap <https://openstreetmap.org>`_. Only geo coordinates
are transmitted.

For reverse image lookups (similar image search), object detection and face
recogintion, HomeGallery uses the
its own public API at ``api.home-gallery.org``. This public API supports
low powered devices to run HomeGallery such as the SoC Raspberry PI.
All low resolution preview images are send to this public API by default.
No images or privacy data are kept from this API.

The API can be configured and ran also locally or as Docker container. See
:ref:`api-server` section for details.

How many data is stored in the storage directory?
-------------------------------------------------

The media storage holds all extracted meta data, preview images and videos.
See :ref:`data structure <data-structure-storage>` for details.
Videos are transcoded to a web friendly 702p video and preview images are
resize to full HD (1920x1080) and lower.

Therefore, the memory consumption on the storage directory depends on the
amount of images and videos. A rule of thumb would be 20% of your
original space.

How long does the initial setup take?
-------------------------------------

It depends on the amount and images and videos. On the initial
run all preview images and videos needs to be calculated. See
design desisions for :ref:`precalculations <design-decision-prerendering>`.

For images you have think in hours or days. For videos in days
or weeks. For that reason it is a good idea to start with some
sub directories and exclude video files like AVI, MOV, MPEG or MP4
files. Process further partent directories or videos later
for you need and patience.

Use following exclude file to process only image files (JPG and PNG):

.. literalinclude:: files/only-images.exclude
    :language: text

Due the internal structure already processed previews are not
recalculated later.

Why is a progress indicator for all files is missing?
-----------------------------------------------------

TL;DR the current non-progress state is on purpose. The recommended
import is the initial (progressive) import via ``./gallery run import --initial``.

Looong version: Progress information on long taking processes like importing a
gallery data is awesome. The assumption is that a user imports 50k - 100k media
files (0,5 TB - 1 TB data) with some videos and wants to see as early as possible
some pictures on the gallery.

However due the architecture of HomeGallery the files need to be indexed and the file
checksums need to be calculated which are used as unique file ids. Afterwards the
preview files and web compatible video previews need to be calculated. As final
step the database is build. So there is no intermediate step to shortcut to present
pictures in the browser.

If the whole media archive is processed at once it is time consuming for checksums,
image and video previews. As mentioned, the image previews will take
days, the video previews will take weeks depending on your amount of pictures,
videos and host machine.

Since the assumed standard user wants to see early results, the recommended import
of an archive should be incremental (or by chunks) via
``./gallery run import --initial``. Than the files are indexed and processed in chunks
and the database is build successive. The gallery will reload automatically when the
database changes. So the user sees some pictures early while the whole archive
import is in progress for weeks.

The internals of this progress is done via the indexer and the ``--add-limits``
parameter listed by ``./gallery index -h`` (see also the
`source comments <https://github.com/xemle/home-gallery/blob/master/packages/index/src/limit-filter.js>`_).
This uses the gallery characteristic that the gallery knows only the files, which
are indexed. So the indexer stops after adding some new files (afterwards these new
files are processed, the database is updated, new files are indexed and so on and
so forth).

How does the image similarity work?
-----------------------------------

The image similarity or reverse image search works through a pretrained
image net of machine learning. It uses pretrained Google's mobilenet to
extract an similarity vector which is than compared agains other images.

In detail is uses the embeddings layer for the similarity vector and optimizes
(or reduces the size) for the browser. The embeddings layer is retrieved
via the API server as extractor step. When the database is build, the
raw vector is reduced in data - in number of data points and in precision.

The exact algorithm where discovered in an investigation/analysis
phase. As result it is more important to have data points than the precision.
Therefore, the initial vector of 1024 floats is reduces to a third
of 341 numbers with a 2 bit precision. These bits are packed to bytes and
encoded in base64 to a JSON string with 116 chars like:

.. code-block::

    EGUUohiokFBoKUWwUhsFdRjUAEQQARElJZQmDlVCK6lgwsTGm2SRUYYEQGgAVRCCVEoQJFABVbJRSEIoQ1rpHDQUCIAWnVQRTlVSFIlDZQUEhLiFgAA=

With this procedure it is possible to reduce 1.3 GB raw vector data of
100,000 images to about 11 MB with some quality reduction but better browser
support.

The comparison is done by the cosine similarity from a seed similarity
vector against all other images. In modern browsers - even on mobile
devices the similarity comparison takes less than 500ms for about 100,000
images.

Exact limits, factors or scaling should be extracted from the source.

Do I need to delete the storage directory?
------------------------------------------

Usually you do not need to delete any file in the storage directory.
In the strorage directory all previews and meta data are kept.
During processing existing previews and meta data are recognized
and skiped.

You should only delete this storage if you migrate the storage
or you like to delete the HomeGallery files (e.g. switching to another
gallery solution).

Entry files cache is missing
----------------------------

If your stumble across a *Entry files cache is missing* error like:

.. code-block::

    extract:meta Entry files cache for
    4d6c610:Pictures:Themen\Wallpaper\marriage toni\RIMG0056.JPG is missing.
    Read files and meta from storage +4ms

The reason for this message is that the extractor most
probably did not processed this file. This might happen due an
incomplete or broken full media processing.

Are Raw Images supported?
-------------------------

If a RAW image file (or unsupported image format) has a JPG sidecar
file, HomeGallery will use this JPG file for preview.
E.g. ``IMG_1234.cr3`` has a JPG sidecar file ``IMG_1234.jpg`` or
``IMG_1234.cr3.jpg`` than preview images are generated from
``IMG_1234.jpg`` (or ``IMG_1234.cr3.jpg``).

If no JPG sidecar is available, HomeGallery can extract embedded
preview images from the RAW image file through ``exiftool`` if
available. These previews have mostly a lower resolution than the
original image.

.. note::
    It is assumed that disk space is not an issue for the usecase of
    HomeGallery. It is recommended to generate JPG sidecars by your
    own to have good image preview for your raw images.
