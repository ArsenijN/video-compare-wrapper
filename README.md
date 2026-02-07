# video-compare-wrapper
[video-compare](https://github.com/pixop/video-compare/) tool wrapper, adding support for drag'n'drop from SMB drives on KDE systems. Made from the [own issue](https://github.com/pixop/video-compare/issues/131#issuecomment-3863643218)

# About
It's just simple wrapper to add the SMB support. Details about how to do that is pending

Basically you will need to or compile the video-compare from source with the right FFmpeg config, or compile FFmpeg from source, installing the video-compare from brew and editing PATH to add the own build of FFmpeg, or... (There's a lot of configurations how to do that, I'll leave what I did - unideal but works)

1. Install video-compare from brew: ``
2. Get an latest source of FFmpeg (warning! you may need specific versions of libs that included in FFmpeg, you will see what exactly on first attempt to run the video-compare after all the things done): `wget https://ffmpeg.org/releases/ffmpeg-8.0.1.tar.xz && tar xf ffmpeg-8.0.1.tar.xz && cd ffmpeg-8.0.1`
3. Configure the FFmpeg with `--enable-libsmbclient` option to enable SMB support: `./configure     --enable-libsmbclient     --enable-gpl     --enable-version3     --enable-shared     --prefix=/usr/local`
4. Compile FFmpeg with `make -j$(nproc)`
5. Install it: `sudo make install`
7. ~~Copy to bin folder~~
8. Add new FFmpeg build to the PATH: `(automatically handled? need a test on fresh system to avoid weird things to happen)`
9. ~~Delete FFmpeg from dependencies in brew (actually just change the PATH~~
10. Try to run video-compare

If after everything that was made, video-compare complaints about the missing libs - do from the 2nd to 10th steps again, but use the FFmpeg version that comes with those libs
