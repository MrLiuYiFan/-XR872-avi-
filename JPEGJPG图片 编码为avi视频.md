# JPEG/JPG图片 编码为avi视频

## 1：简介

​		在XR872可视门铃项目中，需要本地摄像头拍照上传视频到第三方服务器，因为本地没有摄像功能，所以只能本地编码为视频格式。（jpeg为cpu自动压缩编码）

​		本来是要编辑为mjpeg格式的视频，可视奈何找不到格式编码资源，所以只能寻找可行的、可移植的demo。

本demo来源与github（）

​		本地图像数据在sd卡，即全志平台已经移植好FatFs文件系统（官网 http://elm-chan.org/fsw/ff/00index_e.html ）。（f_open、f_read等）

## 2：AVI具体格式

​		avi视频格式。			

​		详情见demo 。

## 3：问题

1：移植过程中  读取图片数据  f_putc 莫名其妙就加上了 ‘\r’。

​	内部函数宏定义 搞得鬼

```c
static
void putc_bfd (		/* Buffered write with code conversion */
	putbuff* pb,
	TCHAR c
)
{
	UINT bw;
	int i;

//此处问题**************************************//
	if (_USE_STRFUNC == 2 && c == '\n') {	 /* LF -> CRLF conversion */
		putc_bfd(pb, '\r');
	}

	i = pb->idx;		/* Write index of pb->buf[] */
	if (i < 0) return;

#if _LFN_UNICODE
#if _STRF_ENCODE == 3			/* Write a character in UTF-8 */
	if (c < 0x80) {				/* 7-bit */
		pb->buf[i++] = (BYTE)c;
	} else {
		if (c < 0x800) {		/* 11-bit */
			pb->buf[i++] = (BYTE)(0xC0 | c >> 6);
		} else {				/* 16-bit */
			pb->buf[i++] = (BYTE)(0xE0 | c >> 12);
			pb->buf[i++] = (BYTE)(0x80 | (c >> 6 & 0x3F));
		}
		pb->buf[i++] = (BYTE)(0x80 | (c & 0x3F));
	}
#elif _STRF_ENCODE == 2			/* Write a character in UTF-16BE */
	pb->buf[i++] = (BYTE)(c >> 8);
	pb->buf[i++] = (BYTE)c;
#elif _STRF_ENCODE == 1			/* Write a character in UTF-16LE */
	pb->buf[i++] = (BYTE)c;
	pb->buf[i++] = (BYTE)(c >> 8);
#else							/* Write a character in ANSI/OEM */
	c = ff_convert(c, 0);	/* Unicode -> OEM */
	if (!c) c = '?';
	if (c >= 0x100)
		pb->buf[i++] = (BYTE)(c >> 8);
	pb->buf[i++] = (BYTE)c;
#endif
#else							/* Write a character without conversion */
	pb->buf[i++] = (BYTE)c;
#endif

	if (i >= (int)(sizeof pb->buf) - 3) {	/* Write buffered characters to the file */
		f_write(pb->fp, pb->buf, (UINT)i, &bw);
		i = (bw == (UINT)i) ? 0 : -1;
	}
	pb->idx = i;
	pb->nchr++;
}


static
int putc_flush (		/* Flush left characters in the buffer */
	putbuff* pb
)
{
	UINT nw;

	if (   pb->idx >= 0	/* Flush buffered characters to the file */
		&& f_write(pb->fp, pb->buf, (UINT)pb->idx, &nw) == FR_OK
		&& (UINT)pb->idx == nw) return pb->nchr;
	return EOF;
}

```

2：视频存储  test.avi 命名 ，第一次f_open,第二次open之后查看hex编码 没有改变但是数据大小改变了。待解决