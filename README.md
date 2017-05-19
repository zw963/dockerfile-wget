# dockerfile-wget
Create a bash function in Dockerfile, and use it to download static compile wget.

Does you use Dockerfile like this:

```Dockerfile

# install wget
RUN  apt-get update && apt-get install -qq -y --no-install-recommends wget && rm -rf /var/lib/apt/lists/*

#####################

# do whatever tasks which need wget to download somethings ...

######################

# remove wget
RUN apt-get purge -y --auto-remove wget

```

This is not the correct way to do this.
If run ``docker history your_image``, you will see

```sh
...
41753cb4a6be        4 hours ago         /bin/sh -c apt-get purge -y --auto-remove ...   445 kB              
1eae68198132        17 hours ago        /bin/sh -c apt-get update && apt-get insta...   34.4 MB
...
```

Because you `install wget` and `remove wget` in seperate step, you got about `34MB` unuseful data
into docker layout, in other word, you image size get bigger.

Another way ``CORRECT`` way to do this is, in every step you need wget, write as followings

```dockerfile
RUN apt-get update && apt-get install -qq -y --no-install-recommends wget && rm -rf /var/lib/apt/lists/* && \
         do-whatever-process-need-wget && \
         RUN apt-get purge -y --auto-remove wget
```

This method is good, but too much redundant code, not perfect.

## Why use bash to download a static compiled wget, and keep use it?

This project is the attempt to do this, following is a example dockerfile

```dockerfile
FROM ruby:2.3-slim

MAINTAINER Billy Zheng(zw963) <vil963@gmail.com>

RUN bash -c 'function __wget() { \
    local URL=$1; \
    read proto server path <<<$(echo ${URL//// }); \
    local SCHEME=${proto//:*} PATH=/${path// //} HOST=${server//:*} PORT=${server//*:}; \
    [[ "${HOST}" == "${PORT}" ]] && PORT=80; \
    exec 3<>/dev/tcp/${HOST}/${PORT}; \
    echo -en "GET ${PATH} HTTP/1.1\r\nHost: ${HOST}\r\nConnection: close\r\n\r\n" >&3; \
    local state=0 line_number=0; \
    while read line; do \
        line_number=$(($line_number + 1)); \
        case $state in \
            0) \
                echo "$line" >&2; \
                if [ $line_number -eq 1 ]; then \
                    if [[ $line =~ ^HTTP/1\.[01][[:space:]]([0-9]{3}).*$ ]]; then \
                        [[ "${BASH_REMATCH[1]}" = "200" ]] && state=1 || return 1; \
                    else \
                        printf "invalid http response from '%s'" "$URL" >&2; \
                        return 1; \
                    fi; \
                fi; \
                ;; \
            1) \
                [[ "$line" =~ ^.$ ]] && state=2; \
                ;; \
            2) \
                echo "$line"; \
        esac \
    done <&3; \
    exec 3>&-; \
}; \

__wget http://202.56.13.13/base64_encoded_static_wget |base64 -d > /usr/local/bin/wget; \

if [ -s /usr/local/bin/wget ]; then \
    chmod +x /usr/local/bin/wget; \
else \
    echo "Network is not connected?" >&2; \
    exit 1; \
fi;'


CMD ["ruby"]
```

When it build, we can see ``docker history image_name``

```sh
9d95c852997c        31 seconds ago      /bin/sh -c bash -c 'function __wget() {     l   618.3 kB 
```

Oh, From ``445 kB + 34.4 MB`` reduce to `618.3kb`, it just the size of `/usr/local/bin/wget`, which
we got from [minos-static](https://github.com/minos-org/minos-static) project, what i does is just


- download static compiled wget, and pack with ``upx`` (size from 1.5M reduce to 604kb)
- encode use base64, for transmit from internet.
- create ``__wget`` function in Dockrfile, for a more complete example, read [this StackExchange answer](https://unix.stackexchange.com/a/83927/148127).
- use ``_wget`` function to download wget.

















