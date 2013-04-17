# openstack netease ppa源使用方法

## 将下面的公钥保存为openstack.asc

    root# cat > openstack.asc << EOF
    -----BEGIN PGP PUBLIC KEY BLOCK-----
    Version: GnuPG v1.4.12 (GNU/Linux)

    mQENBFEsg/EBCACjfp2HBpevoZ7GfZp9b4UzoU1sMVpro0fmzYBnRuYBHn2JUC3F
    wmYzPwn9oH4fSDc1XyBwZLLqMcPyEwGFz2UayEAKxkh41gYmCszmKjLaFDNJPVnw
    cWiAGCjkNEcCN5s20d6CBsLM9uefMW2/QRlXC0dIN5caKNURhE7qzKW9CT6cSiap
    axKQWSMqUqCiI1xgeb1HX+FQ+/HVo3oWjFtqupd9gU8wD8m1Dz++Aqr/X3g0ZvWC
    3vCVAnZO4p7ULH/IZWkvNkc8ZIysCJyfIPV76xJSjBVsG2SNMWvQurwWMW6OuACN
    oLijCXUytdxw9WXQHbpgA59VviPqX7Ykb4Y1ABEBAAG0Nk5ldEVhc2UgT3BlbnN0
    YWNrIFBQQSBTaWduaW5nIEtleSA8c3Rhbi56Z3lAZ21haWwuY29tPokBOAQTAQIA
    IgUCUSyD8QIbAwYLCQgHAwIGFQgCCQoLBBYCAwECHgECF4AACgkQRg7YrjYiVjUj
    1gf/QTBfDfzztri96wbVuq6Num5zdGlJnuzj2aP9fb1yXbWmw4/f9bkaxF0+iNGW
    Fy/QNja0xV6Ny7qkzhaQUflrlixft1dGdmhSDFeBIYKP+aOxaNMKVXx8RB0jUQfk
    CFOJS8O3c3lh0O4JWBMk/m+KF83v2MudZ7fw/SMQp4ktNYVHd7A/udMyAWRd/0Xz
    U8RXzxSFNPfY8JTFnHjENRBZMznJ1im4oqPG6pEg5gN6NAwAHyNYYH7rK8TTEu2Z
    Q911DtNd6uuyaeqhu9pQxgQ0SlMKhQjBVHeM/61FF8zNiQOfucrCc6rTPOZTWCeo
    /uYRZlxN7dRnyI8a7q4+HTZsKLkBDQRRLIPxAQgA5mN2Q+PbeT4PnXTHGJR8f9xF
    p2ghoDPlqrdJu1xfMGe1cMqOXrBeRGSMlJzXwti+NtrJ+WLy1v29uGCviBY9M5g9
    60KwVRrMcaW/pYSEyOJwrpWtxfYRh1YfFLwce+Izg9CipiySq9LNQmdI8GVSWAtU
    bitGAihIbK9s2DvHRu304YzoLp4etSl0Fyk9eTcDVcO3o6LPoMmZM4THrh+EBTMC
    x3iXzNolETrbU1au/u50kBwy15R5Jeko58f248URNwx0b1V6KeHy4UO5YN5RS9Xr
    PwscYRUoFOPPrm4yQw/Jis2HQ/c20N0SHQOphuILyuJZJ5zMLIVgT5c68ky5JQAR
    AQABiQEfBBgBAgAJBQJRLIPxAhsMAAoJEEYO2K42IlY1EsEIAI2C6yIVgoJzK4Ko
    dsj52/3VS/UhbLBl2B7/UDacVgzert9mHAddbVmp386rr2Qr2Y8iinwjRFq/Nk6Q
    pzARblPeda09GDANrKfA2wtkktZ1chTtV/dAljkG4khT13p26nzZwZWialGZ2JRp
    duPXh8tLb04IGnUPCHvWbaG9pEtQiWRgEfOv89MEVcEkAR7B8is0ASmPD30OIQLD
    ElWdLNxdVHhds5dYN5mOS6VR3/jsg6uQRYENmyKUJi/ar7gnjcLjy5vFDvMIp0OF
    7CDQTXU7aeEUcah7Lucz87DoGdjCb5Ey1xYXdSfIo81W1rVh3L/rQ6wH2SKAOZdo
    hHJzfeY=
    =njWV
    -----END PGP PUBLIC KEY BLOCK-----
    EOF


## 添加pgp key到aptitude

运行
    root# apt-key add openstack.asc

然后 apt-key list看一下, 如果看到下面的内容说明key导入成功了, 可以进行下面得流程

    pub   2048R/36225635 2013-02-26
    uid                  NetEase Openstack PPA Signing Key <stan.zgy@gmail.com>


## 修改/etc/apt/sources.list, 加入我们的ppa源

    root# echo "deb http://114.113.199.8:82/debian wheezy main contrib non-free" >> /etc/apt/sources.list


## 更新aptitude index

    root# aptitude update


## 确认版本, 安装需要的package

以python-nova为例

    root# aptitude show python-nova

输出的结果中找到这行

    Version: 2012.2+netease.m4.3.4-1

确认是否是想要的package版本. 如果version中有netease, 说明是我们自己的package, 可以直接安装了.
PS: 只有我们自己做了修改和开发的package, version中才会有netease. 比如nova, glanceclient等

    root# aptitude install python-nova

如果版本号不对, 可能是添加了更新的源(experimental)导致, 需要手工指定版本号安装.

查看可用的package list

    root# apt-cache showpkg python-nova

在输出的结果中找到provides list

    Provides:
    2012.2+netease.m4.3.4-1 -
    2012.1.1-13 -

## 指定合适的版本安装

    root# aptitude install python-nova=2012.2+netease.m4.3.4-1
