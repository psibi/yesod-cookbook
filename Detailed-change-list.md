This page mostly focuses on detailed views of breaking changes, not necessarily covering new features. For a higher-level view, see [[Changelog]].

# Not yet released Yesod 1.2

* Removed a number of deprecated functions from Yesod.Widget. These can be replaced with `toWidget`, `toWidgetHead`, `toWidgetBody`, and `toWidgetMedia`. `toWidgetMedia` was added in 1.2 and replaces `addLuciusMedia` and `addCassiusMedia` as a means of adding CSS that only applies to specific media types.

# 2012-12-27

* yesod-core 1.1.7 has introduced a new means of creating session backends (clientSessionBackend2) and deprecated the old one. Going forward, the old method will be removed, which will allow us some significant performance optimizations. See: https://github.com/yesodweb/yesod/pull/418
* hamlet 1.1.3.1 now gives you an error message when a class name contains a hash, to avoid the problem of IDs accidentally being slurped up by class names. See: https://github.com/yesodweb/shakespeare/issues/75