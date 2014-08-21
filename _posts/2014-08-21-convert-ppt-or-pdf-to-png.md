---
layout: post
title: "将 PPT/PDF 转为图片(Convert PDF or PPT to Image)"
description: ""
category: 
tags: []
---

{% highlight ruby %}
def show
  @widget = Widget(params[:id])
  respond_to do |format|
    format.html # show.html.erb
    format.json { render json: @widget }
  end
end
{% endhighlight %}

使用 `docsplit` 可以将 PDF/PPT 转化为 Image(JPEG, PNG等)。事实上 [Docsplit](http://documentcloud.github.io/docsplit/) 是一个很强大的工具：Docsplit is a command-line utility and Ruby library for splitting apart documents into their component parts. Under the hood, Docsplit is a thin wrapper around the excellent GraphicsMagick, Poppler, PDFTK, Tesseract, and LibreOffice libraries. 

- 将文档的每一页保存为一张图片

    ```
    docsplit images example.pdf
    docsplit images docs/*.pdf --size 700x,50x50 --format gif --pages 3,10-15,42

    Docsplit.extract_images('example.doc', :size => '1000x', :format => [:png, :jpg])
    ```

- Extract the complete UTF-8-encoded plain text of a document to a single file

    ```
    docsplit text path/to/doc.pdf --pages all

    docs = Dir['storage/originals/*.doc']
    Docsplit.extract_text(docs, :ocr => false, :output => 'storage/text')
    ```

- Burst apart a document into single-page PDFs

    ```
    docsplit pages path/to/doc.pdf --pages 1-10

    Docsplit.extract_pages('path/to/presentation.ppt')
    Docsplit.extract_pages('doc.pdf', :pages => 1..10)
    ```

- Convert documents into PDFs
    Any type of document that LibreOffice can read may be converted.

    The first time that you convert a new file type, LibreOffice will lazy-load the code that processes it — subsequent conversions will be much faster.

    ```
    docsplit pdf documentation/*.html
    Docsplit.extract_pdf('expense_report.xls')
    ```

- author, date, creator, keywords, producer, subject, title, length

    Retrieve a piece of metadata about the document. The docsplit utility will print to stdout, the Ruby API will return the value.

    {% highlight bash %}
    docsplit title path/to/stooges.pdf
    => Disorder in the Court
    Docsplit.extract_length('path/to/stooges.pdf')
    => 36
    {% endhighlight %}


开始转 PPT 为 PNG
------
需要完成的工作是，客户上传一个 PPT 文件，然后 Rails 将每张 Slide 转为一个 PNG 图片。

使用命令 `docsplit images example.ppt` 可以完成，需要安装一种 Office (OpenOffice/LiberOffice)，我们的服务器是 Ubuntu ，`apt-get install liberoffice`轻松安装 Office 。但是生成的图片中，中文字体都是一个个的小方块，这是因为 Ubuntu 服务器上没有中文字体，无法呈现出中文。安装中文字体：

    apt-get install ttf-wqy-microhei ttf-wqy-zenhei xfonts-wqy ttf-arphic-ukai

再次执行转化命令，应该就可以正确地显示中文了。

Linux 系统的字体设置很复杂，至今我也没太搞明白，可以从下面网站下载更多字体： http://wenq.org/cloud/fcdesigner_local.html

PPT 中使用的字体，在 Ubuntu 服务器上可能没有安装，这样会 Fallback 到使用系统可用的中文字体，导致生成的图片与原 PPT 中字体不一样。有两种方式可以解决这个问题：
- 安装缺少的字体，但是上传的 PPT 中可能使用各式各样的字体，即使安装再多的字体也无法完全避免这个问题;
- 要求用户在自己电脑上使用 PPT 软件将要上传的 PPT 文件导出为 PDF 格式，然后上传 PDF 文件，服务器将 PDF 文件转为 PNG 图片；

第二种方式可以完美地在生成的图片中呈现原 PPT 的内容，这是因为 PDF(Portable Document Format) 内嵌了文档中使用的字体. Each PDF file encapsulates a complete description of a fixed-layout flat document, including the text, __fonts__, graphics, and other information needed to display it.


References
------------
- [ ArchLinux Wiki 上关于字体的介绍](https://wiki.archlinux.org/index.php/Fonts_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
- [ Ubuntu Wiki 上关于字体的介绍](http://wiki.ubuntu.com.cn/%E5%AD%97%E4%BD%93)
- [How to convert a ppt to a pdf from the command line?](http://askubuntu.com/questions/11130/how-can-i-convert-a-ppt-to-a-pdf-from-the-command-line)
- [unoconv: Convert between any document format supported by OpenOffice](http://dag.wiee.rs/home-made/unoconv/)
