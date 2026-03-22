# psplash_wangyonglin_loongify
    开机动画

```
apt-get install build-essential libncurses5-dev 
apt-get install autoconf
apt-get install libtool
apt-get install gettext
apt-get install libglib2.0-dev
apt-get install libgtk2.0-dev
sudo apt-get install libgdk-pixbuf2.0-dev

root@iZuf6duhz4f6oour5xv5anZ:/opt/psplash_flying_logo#  automake --add-missing
Makefile.am: error: required file './AUTHORS' not found
Makefile.am: error: required file './ChangeLog' not found
Makefile.am: error: required file './NEWS' not found
root@iZuf6duhz4f6oour5xv5anZ:/opt/psplash_flying_logo#   touch NEWS README ChangeLog AUTHORS
```

### Buildroot

    对于Buildroot根文件系统，将psplash 和 psplash-write放到文件系统/usr/bin目录下，编写S00paplash脚本，放到/etc/init.d/目录下，启动运行的第一个脚本。

```
#!/bin/sh 
### BEGIN INIT INFO
# Provides:             psplash
# Required-Start:
# Required-Stop:
# Default-Start:        S
# Default-Stop:
### END INIT INFO

read CMDLINE < /proc/cmdline
for x in $CMDLINE; do
        case $x in
        psplash=false)
		echo "Boot splashscreen disabled" 
		exit 0;
                ;;
        esac
done

export TMPDIR=/mnt/.psplash
mkdir -p $TMPDIR
mount tmpfs -t tmpfs $TMPDIR -o,size=40k

rotation=0
if [ -e /etc/rotation ]; then
	read rotation < /etc/rotation
fi

/usr/bin/psplash --angle $rotation &
```

    psplash 和 psplash-write之间通过管道通信，S00paplash脚本启动了psplash加载了logo和进度条， psplash-write用于向进度条写入数据，修改/etc/init.d/rcS文件

```

#!/bin/sh

# Start all init scripts in /etc/init.d
# executing them in numerical order.
#
for i in /etc/init.d/S??* ;do

     # Ignore dangling symlinks (if any).
     [ ! -f "$i" ] && continue

     case "$i" in
	*.sh)
	    # Source shell script for speed.
	    (
		trap - INT QUIT TSTP
		set start
		. $i
	    )
	    ;;
	*)
	    # No sh extension, so fork subprocess.
	    $i start
	    ;;
    esac
     #15根据启动的脚本个数定
    progress=$(($progress+15))
    if type psplash-write >/dev/null 2>&1; then
        TMPDIR=/mnt/.psplash psplash-write "PROGRESS $progress" || true
    fi
done

```

## 开机动画，开机画面延时

    这一部分的内容，可以自己修改源码，也可以去直接去这个https://download.csdn.net/download/chidaoqi1607/12373339下载。这个老哥也有一篇文章写开机动画的，大家可以参考一下https://blog.csdn.net/chidaoqi1607/article/details/105840790，我这里就不多说了（毕竟人家要收费的）。
    源码部分功能修改

    修改相关源文件
    psplash-config.h

```
/* Text to output on program start; if undefined, output nothing */
#define PSPLASH_STARTUP_MSG ""

/* Bool indicating if the image is fullscreen, as opposed to split screen */
#define PSPLASH_IMG_FULLSCREEN 0

/* Position of the image split from top edge, numerator of fraction */
//图像的位置从上边缘分割，分数的分子
#define PSPLASH_IMG_SPLIT_NUMERATOR 5

/* Position of the image split from top edge, denominator of fraction */
//图像的位置从顶部边缘分割，分数的分母
#define PSPLASH_IMG_SPLIT_DENOMINATOR 6
```

    psplash-colors.h颜色配置文件（背景色 进度条颜色等）

```
/* This is the overall background color */
#define PSPLASH_BACKGROUND_COLOR 0x00,0x00,0x00        //整个背景的颜色

/* This is the color of any text output */
#define PSPLASH_TEXT_COLOR 0xFF,0xFF,0xFF//文本内容

/* This is the color of the progress bar indicator */
#define PSPLASH_BAR_COLOR 0xff,0x00,0x00               //进度条颜色
/* This is the color of the progress bar background */
#define PSPLASH_BAR_BACKGROUND_COLOR 0x00,0x00,0x00    //进度条的背景颜色
```

 ###   设置进度条 高度宽度 psplash_draw_progress（psplash.c）函数中

```
  /* 4 pix border */ //bar.png的(上下左右)边界到进度条区间隔4pix，如果间隔改了，此处应做修改.
  x      = ((fb->width  - BAR_IMG_WIDTH)/2) + 4 ;
  y      = SPLIT_LINE_POS(fb) + 4;
  width  = BAR_IMG_WIDTH - 8; 
  height = BAR_IMG_HEIGHT - 8;
```
###    设置LOGO 进度条的坐标

```
/* Text to output on program start; if undefined, output nothing */
#define PSPLASH_STARTUP_MSG ""  //文本信息

/* Bool indicating if the image is fullscreen, as opposed to split screen */
#define PSPLASH_IMG_FULLSCREEN 1
//为0时：LOGO 居中
//为1时：LOGO 上部分屏居中（height*PSPLASH_IMG_SPLIT_NUMERATOR/PSPLASH_IMG_SPLIT_DENOMINATOR）
//屏分为上下两部分  上部分LOGO居中  下部分进度条居中     
/* Position of the image split from top edge, numerator of fraction */
#define PSPLASH_IMG_SPLIT_NUMERATOR 5

/* Position of the image split from top edge, denominator of fraction */
#define PSPLASH_IMG_SPLIT_DENOMINATOR 6
```

###    去掉进度条 psplash.c
```
/*
   //进度条上面的框框
  psplash_fb_draw_rect (fb, 
            0, 
            fb->height - (fb->height/6) - h, 
            fb->width,
            h,
            PSPLASH_BACKGROUND_COLOR);
*/

#if 0
  /* Draw progress bar border */
  psplash_fb_draw_image (fb, 
             (fb->width  - BAR_IMG_WIDTH)/2, 
             fb->height - (fb->height/6), 
             BAR_IMG_WIDTH,
             BAR_IMG_HEIGHT,
             BAR_IMG_BYTES_PER_PIXEL,
             BAR_IMG_RLE_PIXEL_DATA);
#endif

 // psplash_draw_progress (fb, 0);

```
### 或者将img.h的图片文件中的width和height设置为0
```
#define BAR_IMG_ROWSTRIDE(1400)
#define BAR_IMG_WIDTH(0)
#define BAR_IMG_HEIGHT(0)
#define BAR_IMG_BYTES_PER_PIXEL(4)
```
### .将psplash.sh软链接在根文件系统/etc/rcS.d/目录下，用于开机启动。
拷贝psplash，psplash-write到板卡/usr/bin目录
拷贝S00psplash.sh到板卡/etc/rcS.d目录，实现logo开机自启动
### 参考文章：
    https://blog.csdn.net/al86866365/article/details/82287020
    https://blog.csdn.net/chidaoqi1607/article/details/105840790
    https://neucrack.com/p/91
    https://blog.csdn.net/kunkliu/article/details/81125920
