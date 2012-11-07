---
layout: post
title: Circumventing PDF conversion issues on Manuscript Central
---

# {{ page.title }}
This is a rather short post that is meant to serve as a reminder of what to do when Manuscript Central (MC) rejects the PDF version of your paper generated with `pdflatex`.

By default, `pdflatex` generates PDFs in version 1.5, and to the best of my knowledge, MC accepts PDFs up to version 1.4. If the PDF is of newer version than 1.4, the following error appears in MC:

> Our apologies, but the file(s) your_paper.pdf (PDF Copy) failed to convert to the appropriate pdf and/or html file for review. You may either try uploading the file(s) again in 20 minutes, or contact the Support Team for further assistance. The Support contact information can be found by clicking the "Get Help Now" link in the upper right corner of this page. Read More ...

The easiest way to circumvent this error is to force `pdflatex` to generate a specific version of the PDF. Suppose we settle for version 1.4. This can be achieved like so:

{% highlight latex %}
\documentclass[onecolumn, draftcls, 11pt]{IEEEtran}

\pdfminorversion=4 % tell pdflatex to generate PDF in version 1.4

% The rest of the document...
{% endhighlight %}

Now, MC should gladly accept the PDF version of your paper.
