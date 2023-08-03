---
layout: post
---

I've been trying to make a GUI app using the Fyne framework and Go. An interesting 
thing I thought I'll tackle was to have a keyboard shortcut registered system-wide 
that would pop the app whenever you pressed it. Furthermore, it should work on 
Linux, Windows and Mac.

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

