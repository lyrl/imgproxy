# imgproxy

<img align="right" width="200" height="200" title="imgproxy logo"
     src="https://cdn.rawgit.com/DarthSim/imgproxy/master/logo.svg">

[![Build Status](https://travis-ci.org/DarthSim/imgproxy.svg?branch=feature%2Fdrop-bimg)](https://travis-ci.org/DarthSim/imgproxy)

imgproxy is a fast and secure standalone server for resizing and converting remote images. The main principles of imgproxy are simplicity, speed, and security.

imgproxy can be used to provide a fast and secure way to replace all the image resizing code of your web application (like calling ImageMagick or GraphicsMagick, or using libraries), while also being able to resize everything on the fly, fast and easy. imgproxy is also indispensable when handling lots of image resizing, especially when images come from a remote source.

imgproxy does one thing — resizing remote images — and does it well. It works great when you need to resize multiple images on the fly to make them match your application design without preparing a ton of cached resized images or re-doing it every time the design changes.

imgproxy is a Go application, ready to be installed and used in any Unix environment — also ready to be containerized using Docker.

<a href="https://evilmartians.com/?utm_source=imgproxy">
<img src="https://evilmartians.com/badges/sponsored-by-evil-martians.svg" alt="Sponsored by Evil Martians" width="236" height="54">
</a>

#### Simplicity

> "No code is better than no code."

imgproxy only includes the must-have features for image processing, fine-tuning and security. Specifically,

* It would be great to be able to rotate, flip and apply masks to images, but in most of the cases, it is possible — and is much easier — to do that using CSS3.
* It may be great to have built-in HTTP caching of some kind, but it is way better to use a Content-Delivery Network or a caching proxy server for this, as you will have to do this sooner or later in the production environment.
* It might be useful to have everything built in — such as HTTPS support — but an easy way to solve that would be just to use a proxying HTTP server such as nginx.

#### Speed

imgproxy uses probably the most efficient image processing library there is, called `libvips`. It is screaming fast and has a very low memory footprint; with it, we can handle the processing for a massive amount of images on the fly.

imgproxy also uses native Go's `net/http` routing for the best HTTP networking support.

#### Security

Massive processing of remote images is a potentially dangerous thing, security-wise. There are many attack vectors, so it is a good idea to consider attack prevention measures first. Here is how imgproxy can help:

* imgproxy checks image type _and_ "real" dimensions when downloading, so the image will not be fully downloaded if it has an unknown format or the dimensions are too big (there is a setting for that). That is how imgproxy protects you from so called "image bombs" like those described at  https://www.bamsoftware.com/hacks/deflate.html.

* imgproxy protects image URLs with a signature, so an attacker cannot cause a denial-of-service attack by requesting multiple image resizes.

* imgproxy supports authorization by an HTTP header. That prevents using imgproxy directly by an attacker but allows to use it through a CDN or a caching server — just by adding a header to a proxy or CDN config.

## Installation

There are two ways you can install imgproxy:

#### From the source

1. First, install [libvips](https://github.com/jcupitt/libvips).

  ```bash
  # macOS
  $ brew tap homebrew/science
  $ brew install vips

  # Ubuntu
  $ sudo apt-get install libvips-dev
  ```

  **Note:** Most libvips packages come with WebP support. If you want libvips to support WebP on macOS, you need to install it this way:

  ```bash
  $ brew tap homebrew/science
  $ brew install vips --with-webp
  ```

2. Next, install imgproxy itself:

  ```bash
  $ go get -f -u github.com/DarthSim/imgproxy
  ```

#### Docker

imgproxy can (and should) be used as a standalone application inside a Docker container. It is ready to be dockerized, plug and play:

```bash
$ docker build -t imgproxy .
$ docker run -e IMGPROXY_KEY=$YOUR_KEY -e IMGPROXY_SALT=$YOUR_SALT -p 8080:8080 -t imgproxy
```

You can also pull the image from Docker Hub:

```bash
$ docker pull darthsim/imgproxy:latest
$ docker run -e IMGPROXY_KEY=$YOUR_KEY -e IMGPROXY_SALT=$YOUR_SALT -p 8080:8080 -t darthsim/imgproxy
```

#### Heroku

imgproxy can be deployed to Heroku with the click of the button:

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy)

However, you can do it manually with a few steps:

```bash
$ git clone https://github.com/DarthSim/imgproxy.git && cd imgproxy
$ heroku git:remote -a your-application
$ heroku config:set BUILDPACK_URL=https://github.com/DarthSim/heroku-buildpack-imgproxy.git \
                    IMGPROXY_KEY=$YOUR_KEY \
                    IMGPROXY_SALT=$YOUR_SALT
$ git push heroku master
```

## Configuration

imgproxy is [Twelve-Factor-App](https://12factor.net/)-ready and can be configured using `ENV` variables.

#### URL signature

imgproxy requires all URLs to be signed with a key and salt:

* `IMGPROXY_KEY` — (**required**) hex-encoded key;
* `IMGPROXY_SALT` — (**required**) hex-encoded salt;

You can also specify paths to files with a hex-encoded key and salt (useful in a development environment):

```bash
$ imgproxy -keypath /path/to/file/with/key -saltpath /path/to/file/with/salt
```

If you need a random key/salt pair real fast, you can quickly generate it using, for example, the following snippet:

```bash
$ xxd -g 2 -l 64 -p /dev/random | tr -d '\n'
```

#### Server

* `IMGPROXY_BIND` — TCP address to listen on. Default: `:8080`;
* `IMGPROXY_READ_TIMEOUT` — the maximum duration (in seconds) for reading the entire image request, including the body. Default: `10`;
* `IMGPROXY_WRITE_TIMEOUT` — the maximum duration (in seconds) for writing the response. Default: `10`;
* `IMGPROXY_DOWNLOAD_TIMEOUT` — the maximum duration (in seconds) for downloading the source image. Default: `5`;
* `IMGPROXY_CONCURRENCY` — the maximum number of image requests to be processed simultaneously. Default: `100`;
* `IMGPROXY_MAX_CLIENTS` — the maximum number of simultaneous active connections. Default: `IMGPROXY_CONCURRENCY * 2`;

#### Security

imgproxy protects you from so-called image bombs. Here is how you can specify maximum image dimensions which you consider reasonable:

* `IMGPROXY_MAX_SRC_DIMENSION` — the maximum dimensions of the source image, in pixels, for both width and height. Images with larger real size will be rejected. Default: `4096`;

You can also specify a secret to enable authorization with the HTTP `Authorization` header:

* `IMGPROXY_SECRET` — the authorization token. If specified, request should contain the `Authorization: Bearer %secret%` header;

#### Compression

* `IMGPROXY_QUALITY` — quality of the resulting image, percentage. Default: `80`;
* `IMGPROXY_GZIP_COMPRESSION` — GZip compression level. Default: `5`;

## Generating the URL

The URL should contain the signature and resize parameters, like this:

```
/%signature/%resizing_type/%width/%height/%gravity/%enlarge/%encoded_url.%extension
```

#### Resizing types

imgproxy supports the following resizing types:

* `fit` — resizes the image while keeping aspect ratio to fit given size;
* `fill` — resizes the image while keeping aspect ratio to fill given size and cropping projecting parts;
* `crop` — crops the image to a given size;
* `force` — resizes the image to a given size without maintaining the aspect ratio.

#### Width and height

Width and height parameters define the size of the resulting image. Depending on the resizing type applied, the dimensions may differ from the requested ones.

#### Gravity

When imgproxy needs to cut some parts of the image, it is guided by the gravity. The following values are supported:

* `no` — north (top edge);
* `so` — south (bottom edge);
* `ea` — east (right edge);
* `we` — west (left edge);
* `ce` — center;
* `sm` — smart. `libvips` detects the most "interesting" section of the image and considers it as the center of the resulting image. **Note:** This value is only applicable with the `crop` resizing type.

#### Enlarge

If set to `0`, imgproxy will not enlarge the image if it is smaller than the given size. With any other value, imgproxy will enlarge the image.

#### Encoded URL

The source URL should be encoded with URL-safe Base64. The encoded URL can be split with `/` for your needs.

#### Extension

Extension specifies the format of the resulting image. At the moment, imgproxy supports only `jpg`, `png` and `webp`, them being the most popular and useful web image formats.

#### Signature

Signature is a URL-safe Base64-encoded HMAC digest of the rest of the path including the leading `/`. Here's how it is calculated:

* Take the path after the signature — `/%resizing_type/%width/%height/%gravity/%enlarge/%encoded_url.%extension`;
* Add salt to the beginning;
* Calculate the HMAC digest using SHA256;
* Encode the result with URL-secure Base64.

You can find helpful code snippets in the `examples` folder.

## Source image formats support

imgproxy supports only the most popular image formats of the moment: PNG, JPEG, GIF and WebP.

## Author

Sergey "DarthSim" Aleksandrovich

Many thanks to @romashamin for the awesome logo.

## License

imgproxy is licensed under the MIT license.

See LICENSE for the full license text.
