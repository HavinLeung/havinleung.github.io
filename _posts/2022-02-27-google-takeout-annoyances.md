---
layout: post
title: Google Photos to Apple Photos
usesmathjax: false
---

If this is TLDR, skip to the [bottom](./#the-actual-steps).

Moving from Google Photos to some other photo management software (Apple Photos in my case) is annoyingly difficult.
Sure, you can easily take out all your photos by simply using [Google Takeout](https://takeout.google.com/settings/takeout),
but there's some annoying gotchas.

1. Your media's EXIF data may or may not be what you expect
  - e.g. Some photos I took in 2015 were dated to 2022 in EXIF.
  This would break the photo sorting in other software.
2. For an image called `image.jpg`, Google Photos saves metadata like GPS data, photo timestamp, etc.. into `image.jpg.json` instead of embedding it into `image.jpg`'s exif data
3. The naming of the metadata json files is hilariously inconsistent
  - Just appending `.json` doesn't always work
  - Some files implicitly share a `.json` file with another (e.g. `image.jpg, image-edited.jpg` both correspond to `image.jpg.json`)
  - `img(1).jpg` maps to `img.jpg(1).json` instead of `img(1).jpg.json`
  - etc...
  
It almost feels like Google's intentionally making it subtly difficult to migrate away from Google photos...
  
I googled a bunch to look for a solution and found 2 useful but incomplete pages:

1. [This blog post](https://legault.me/post/correctly-migrate-away-from-google-photos-to-icloud) provides an exiftool
command to take the useful JSON metadata like creation time, GPS coordinates, etc.. and embeds it into the media files
  - Unfortunately they assume that the naming of the json files is consistent
2. [This Github project](https://github.com/mattwilson1024/google-photos-exif) is a node project that populates the `DateTimeOriginal` EXIF field
  - The author [notes](https://github.com/mattwilson1024/google-photos-exif#how-are-media-files-matched-to-json-sidecar-files) the 
  inconsistent JSON naming and has heuristics to work around this
    - The 3 rules are comprehensive, but not complete
  - This only takes fills the `DateTimeOriginal` field and drops useful GPS data
  
If I refine the 2nd page's heuristics a little bit, I can just rename the JSON files to be consistent and use the 1st page's exiftool command!
  
## My turn to try

I wrote a `corresponding_json` function and iterated on it, starting with the simple heuristic of appending `.json` to the media file name.
Iterating a few times, I noticed:

- `image-edited.jpg` generally corresponds to `image.jpg`
- The `.json` files appear to have cap of 51 characters (most of the time)
  - if it's a duplicate file like `image(1).jpg`, then it can actually be 54 characters.
  I.e. the `(1)` didn't count towards their 51 character limit
- Live photos were exported as `image.jpg` and `image.mp4`, with `image.jpg.json` as the metadata file

Accounting for all of this weirdness (source code [here](https://github.com/HavinLeung/google-takeout/blob/master/process.py)),
my `corresponding_json` function was able to find the corresponding json for all ~7,000 files I had taken out.

To make the exiftool command linked above work as expected, I just essentially did 
```python
copy_file(corresponding_json('image.jpg'), 'image.jpg.json')
```

## The actual steps

1. clone [my repo](https://github.com/HavinLeung/google-takeout)
2. run `./process.py <PATH-TO-GOOGLE-TAKEOUT-FOLDER>`
  - If all goes well, you shouldn't get any warnings. If you do, you can tinker with the source code
3. run `exiftool -r -d %s -tagsfromfile "%d/%F.json" "-GPSAltitude<GeoDataAltitude" "-GPSLatitude<GeoDataLatitude" "-GPSLatitudeRef<GeoDataLatitude" "-GPSLongitude<GeoDataLongitude" "-GPSLongitudeRef<GeoDataLongitude" "-Keywords<Tags" "-Subject<Tags" "-Caption-Abstract<Description" "-ImageDescription<Description" "-DateTimeOriginal<PhotoTakenTimeTimestamp" -ext "*" -overwrite_original -progress --ext json <PATH-TO-GOOGLE-TAKEOUT-FOLDER>`


