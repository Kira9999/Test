##Android H5容器 WebView优化

webView.webViewClient = object : WebViewClient() {
    override fun shouldInterceptRequest(view: WebView?, request: WebResourceRequest?): WebResourceResponse? {
        if (request != null) {
            val url = request.url.toString()
            val responseFuture = CompletableFuture<WebResourceResponse?>()

            // 异步处理
            GlideImgCacheManager.interceptRequest(view, url) { response ->
                responseFuture.complete(response)
            }

            // 等待异步处理完成并返回结果
            return responseFuture.get()
        }
        return super.shouldInterceptRequest(view, request)
    }
}

object GlideImgCacheManager {
    private const val TAG = "GlideCache"
    private val CACHE_IMG_TYPE: HashSet<String> = hashSetOf("png", "jpg", "jpeg", "bmp", "webp")

    fun interceptRequest(webView: WebView?, url: String?, callback: (WebResourceResponse?) -> Unit) {
        if (url.isNullOrEmpty()) {
            callback(null)
            return
        }

        val extension = MimeTypeMap.getFileExtensionFromUrl(url).toLowerCase()
        if (extension !in CACHE_IMG_TYPE) {
            callback(null)
            return
        }

        Timber.d("开始 glide cache img ($extension), url: $url")
        val startTime = System.currentTimeMillis()

        Glide.with(webView!!)
            .asBitmap()
            .diskCacheStrategy(DiskCacheStrategy.ALL)
            .load(url)
            .into(object : CustomTarget<Bitmap>() {
                override fun onResourceReady(resource: Bitmap, transition: Transition<in Bitmap>?) {
                    val inputStream = getBitmapInputStream(resource, Bitmap.CompressFormat.JPEG)
                    val costTime = System.currentTimeMillis() - startTime

                    if (inputStream != null) {
                        Timber.d("glide cache img($costTime ms): $url")
                        callback(WebResourceResponse("image/jpg", "UTF-8", inputStream))
                    } else {
                        Timber.e("glide cache error.($costTime ms): $url")
                        callback(null)
                    }
                }

                override fun onLoadCleared(placeholder: Drawable?) {
                    // Do nothing
                }

                override fun onLoadFailed(errorDrawable: Drawable?) {
                    Timber.e("Glide cache load failed: $url")
                    callback(null)
                }
            })
    }

    private fun getBitmapInputStream(bitmap: Bitmap, compressFormat: Bitmap.CompressFormat): InputStream? {
        return try {
            val byteArrayOutputStream = ByteArrayOutputStream().apply {
                bitmap.compress(compressFormat, 80, this)
            }
            ByteArrayInputStream(byteArrayOutputStream.toByteArray())
        } catch (t: Throwable) {
            Timber.e(t, "Error converting bitmap to InputStream")
            null
        }
    }
}
