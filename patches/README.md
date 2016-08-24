# Performance improvements for the `setup:static-content:deploy`

These patches are not ready for public consumption, nor are generally in a good shape, they are just exploring where the big performance gains are.

The biggest gain is in caching the less preprocess output keyed by its input (a hackish version of being https://github.com/damz/magento-cloud/blob/static-content-patches/patches/framework/0001-cache-less-output.patch).

This results in massive gains across the board, as some less files are exactly the same between themes, and all less files are always exactly the same between locales.

The second biggest gain is actually in the bundle partitionner (https://github.com/damz/magento-cloud/blob/static-content-patches/patches/framework/0003-bundle-maximum-part-bytes.patch and https://github.com/damz/magento-cloud/blob/static-content-patches/patches/framework/0002-disable-bundle-partitionner.patch). That thing is hopeless in its current form: it forces each part to be generated over and over again (a massive JSON encoding + some templating), which is not cheap neither on CPU nor on memory. Also, I am not sure what the point of it is, and it actually has a configuration option. For now, I just disabled it completely.

The rest is mostly avoiding doing things several times _in the same run_:

 * The scope code resolution in `Magento/App/Config/ScopePool` is super not cheap, and would greatly benefit from some caching (that's a general Magento 2 performance improvement, I suppose): https://github.com/damz/magento-cloud/blob/static-content-patches/patches/framework/0006-config-cache-scope-code.patch
 * The check for `Minification:isEnabled` is actually extremely visible in profiling, and just never changes, so also benefits from in-request caching: https://github.com/damz/magento-cloud/blob/static-content-patches/patches/framework/0004-asset-cache-minification-flag.patch
 * The preprocess on each asset is actually called at least twice, once via the getSourceFile() path (while trying to figure out if an asset is valid), the other via the getContent() path (possibly several times). This is a hack to short circuit it the second time: https://github.com/damz/magento-cloud/blob/static-content-patches/patches/framework/0005-asset-cache-preprocess.patch
 * And finally, all the themes are loaded a few thousand times from disk at each run for no reason, and `getThemeByFullPath` greatly benefit from static caching: https://github.com/damz/magento-cloud/blob/static-content-patches/patches/module-theme/0001-theme-fullpath-cache.patch
