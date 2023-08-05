---
layout: post
---

I've been working on a [GUI app](https://github.com/bogdanbojan/macaw) that uses the Fyne[^fn1] framework and Go. An interesting 
thing I thought I'll tackle was to have a keyboard shortcut registered system-wide 
that would pop the app whenever you pressed it. Furthermore, it should work on 
Linux, Windows and Mac.

The main options that I found for cross-compiling:
- use `go build` - if you don't depend on external C libraries 
- use `fyne-cross`
- use `zig`

Note: When compiling for darwin, you will need the MacOS SDK on your VM. 

You can get the MacOS SDK:
- from [apple's website](https://developer.apple.com/download/)
- programatically


{% highlight docker %}
# programatically
ENV OSX_SDK="MacOSX11.3.sdk"
ENV OSX_SDK_URL="https://github.com/joseluisq/macosx-sdks/releases/download/11.3/${OSX_SDK}.tar.xz"

RUN curl -sSL "$OSX_SDK_URL" -o "/$OSX_SDK.tar.xz"
RUN mkdir /osxsdk && tar -xf "/$OSX_SDK.tar.xz" -C "/osxsdk"

{% endhighlight %}
---

From these options, the problem that I usually encountered after building the 
binaries was related to the hotkeys. They were not registered correctly. In the 
go files, I used the `go:build` tag in order to target different systems. The 
configuration was not properly taken into account when using `fyne-cross`. 
Therefore, I had to resort to another option.

Zig worked brilliantly[^fn2]. You could
easily put a Zig application into a Dockerfile and directly use `FROM scratch`.
Really portable that way. That's because it provides a zero-dependency, 
drop-in C/C++ compiler that supports cross-compilation out-of-the-box.

{% highlight docker %}
# darwin
RUN CGO_ENABLED=1 \
    GOOS=darwin \
    GOARCH=amd64 \
    CGO_LDFLAGS="-mmacosx-version-min=${MACOS_MIN_VER} --sysroot ${MACOS_SDK_PATH} -F/System/Library/Frameworks -L/usr/lib" \
    CC="zig cc -mmacosx-version-min=${MACOS_MIN_VER} -target x86_64-macos-gnu -isysroot ${MACOS_SDK_PATH} -iwithsysroot /usr/include -iframeworkwithsysroot /System/Library/Frameworks" \
    CXX="zig c++ -mmacosx-version-min=${MACOS_MIN_VER} -target x86_64-macos-gnu -isysroot ${MACOS_SDK_PATH} -iwithsysroot /usr/include -iframeworkwithsysroot /System/Library/Frameworks" \
    go build -ldflags="-w -buildmode=pie" -trimpath -o dist/darwin-amd64/macaw-darwin-amd64 ./cmd/macaw/main.go

{% endhighlight %}


{% highlight docker %}

# windows
RUN CGO_ENABLED=1 \
    GOOS=windows \
    GOARCH=amd64 \
    CC="zig cc -target x86_64-windows-gnu" \
    CXX="zig c++ -target x86_64-windows-gnu" \
    go build -trimpath -ldflags='-H=windowsgui' -o dist/windows-amd64/macaw-windows-amd64.exe ./cmd/macaw/main.go

{% endhighlight %}


{% highlight docker %}

# linux
RUN CGO_ENABLED=1 \
    GOOS=linux \
    GOARCH=amd64 \
    go build -o dist/linux-amd64/macaw-linux-amd64 ./cmd/macaw/main.go

{% endhighlight %}


---

References:

[^fn1]: [https://developer.fyne.io/started/cross-compiling](https://developer.fyne.io/started/cross-compiling)
[^fn2]: [https://lucor.dev/post/cross-compile-golang-fyne-project-using-zig/](https://lucor.dev/post/cross-compile-golang-fyne-project-using-zig/)
