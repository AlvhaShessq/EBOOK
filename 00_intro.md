# DAX Patterns — 2nd Edition
**Authors:** Alberto Ferrari & Marco Russo  
**Publisher:** SQLBI Corp., 2020  
**ISBN:** 978-1-7353652-0-6  
**Samples:** www.daxpatterns.com

---

## Introduction

DAX Patterns
SECOND EDITION

The most comprehensive collection of
ready-to-use solutions in DAX for Power BI,
Analysis Services, and Power Pivot.

Alberto Ferrari
Marco Russo
Copyright © 2020 by Alberto Ferrari and Marco Russo

All rights reserved. This publication is protected by copyright, and
permission must be obtained from the publisher prior to any
prohibited reproduction, storage in a retrieval system, or
transmission in any form or by any means, electronic, mechanical,
photocopying, recording, or likewise. Microsoft and the trademarks
listed at www.microsoft.com/en-
us/legal/intellectualproperty/trademarks/usage/general are
trademarks of the Microsoft group of companies. All other marks are
property of their respective owners.

The example companies, organizations, products, domain names,
email addresses, logos, people, places, and events depicted herein
are fictitious. No association with any real company, organization,
product, domain name, email address, logo, person, place, or event
is intended or should be inferred.

This book expresses the author’s views and opinions. The
information contained in this book is provided without any express,
statutory, or implied warranties. Neither the authors, the publisher,
nor its resellers, or distributors will be held liable for any damages
caused or alleged to be caused either directly or indirectly by this
book.


Publisher / Editorial Production: SQLBI Corp., Las Vegas,
NV, United States
Authors: Alberto Ferrari, Marco Russo
Copy Editor: Claire Costa
Technical Editors: Daniil Maslyuk, Sergio Murru
Cover Designer: Daniele Perilli

ISBN: 978-1-7353652-0-6

Library of Congress Control Number: 2020912594


All the samples and files used in this book are available on
www.daxpatterns.com

All the code in this book has been formatted with
www.daxformatter.com
Introduction
 At SQLBI we have a beautiful job: we are world-wide trainers and
 consultants. We meet thousands of people all over the world
 every year: a crowd of very diverse persons, sharing the same
 passion for Business Intelligence and DAX. We are asked to solve
 scenarios of various complexity by our students and customers.
   Say a student approaches you because they need to compute
 the number of new customers for their report. You solve the
 problem once, twice, three times… And at some point, you feel
 that the next time you need to answer the same question, you
 would love to have a ready-to-use solution. This is the reason we
 started the daxpatterns.com website in 2013. We started
 collecting patterns that repeat themselves. We created a
 collection of DAX formulas aimed at solving the most frequently-
 asked questions we receive. At that time, the goal was not to write
 a new book. Instead, our goal was to create some sort of memory
 bank for the solutions we would find. We thought we would be the
 main users of our own website.
   As is often the case, real-life does not go according to plan. This
 time, for the better. The website had a tremendous success.
 Users downloaded the samples and achieved two different goals:
 they found a ready-to-use solution to their problems, and they
 improved their DAX skills based on the formulas we authored.
 Because of the different file formats, we included samples for
Excel 2010 and Excel 2013 – the latter still works with later
versions of Excel. Eventually, we collected the content of the
website into a book. That was the first edition of DAX Patterns. It
was at the end of 2015. At the time, we had not yet published the
first edition of The Definitive Guide to DAX. Therefore, we
included a short introduction to DAX in the DAX Patterns book.
 Many things changed over the following five years. DAX evolved
with many useful features. Most importantly, Power BI hit the
market and the number of users adopting DAX grew at an
exponential rate. Today, most of the DAX users create a Power BI
solution. When we published the first edition of this book, Power
BI had not even been announced yet.
 During these five years, the process of collecting patterns
continued. We met more students, we solved more problems, we
also got better and better at DAX. Plus, we now had thousands of
users who were able to provide feedback on previous patterns.
Studying user comments gave us a better picture of what our
readers needed. In parallel, we went on to publish two editions of
The Definitive Guide to DAX. At that point, there was no longer a
reason to be teaching DAX in a book about patterns.
 Long story short, it started to make a lot of sense to author a
new version of both the DAX Patterns website and book. We
rolled up our sleeves and created the book you are reading right
now.
  We did not use any of the content from the previous book. We
wanted a fresh start. The entire library of code is rewritten from
scratch, using the latest DAX and Power BI features and adapting
the code to Excel 2019 when necessary.
 In this new edition we made several choices:
We greatly increased the share of the book dedicated to
time intelligence calculations. Time intelligence is by far the
most widely studied topic. Therefore, it made sense to
increase the number of time-related calculations and
patterns.
Similarly, the New and returning customers pattern was an
absolute hit. We gave that pattern a bigger share of the
book as well, increasing the number of formulas and
models to compute new and returning customers.
We increased the number of patterns, adding several that –
in our experience – are likely to be useful to our readers.
We decided to cut out a few patterns. For example, the
chapter about statistical calculations was useful back in
2015, because of the lack of statistical functions in DAX.
Since then, DAX introduced many new functions to
compute the formulas that were explained in that chapter.
There is no need for that content in 2020.
We no longer provide code snippets. In the previous book,
most of the code was shown including placeholders for the
columns that readers were likely to change. We no longer
do that. We show code that works, because you often have
to adapt the data model and other details in the formula. We
felt this would make the code more readable and easier to
use and to adapt to your model.
We optimized every single formula. All the code you see in
these patterns has been thoroughly reviewed for
performance. This is not to say that these patterns are the
very best. They are the best we could come up with. If you
can make the code better and faster, let us know! The
      comment sections on the website are the right place to
      provide your feedback.
      We created a Power BI and an Excel version of each
      sample file. In the book, we include pictures of Power BI
      reports showing the results of the code, but the examples
      you can download are available in both formats: Power BI
      and Excel.
      We improved the readability of the eBook version of DAX
      Patterns. This meant keeping the code formatting intact
      regardless of the eBook reader size.


Why we published this book
If you are wondering what the differences may be between the
content of this book and the content published on
daxpatterns.com, we want to assure you that there are no
differences. Should you buy the book to obtain extra content? No.
The access to the web site is free, where you can read the same
content as what you will find in this book and download the
sample files for free.
 That said, if you enjoy having an offline copy of the patterns, if
you enjoy having a printed version, if you would like to have it in
your eBook collection, then you should purchase it. This way, you
help us keep the business up and running. We were surprised
with the number of people who purchased the first edition. This
motivated us to further invest into this new version of the website
and the pattern. We hope the process will continue!
  By visiting the daxpatterns.com website, you will also see that
we have recorded a video for each pattern. This is where we go
into more depth on how to use the patterns and how the formulas
work. These videos are for sale. You can buy all of them, or just
the pattern you want to study more. It is an additional service that
many people have been asking for; we know some prefer the
book, some prefer the video, and many people want both!

How to use this book
What will you find in this book? Each standalone chapter covers a
separate pattern and can be read without having read the others.
You can read the Currency conversion pattern without having ever
looked at the Basket analysis, or at any of the time-related
calculations.
 Each chapter about a pattern starts with a brief description of the
business scenario; it then goes into a more complete description
of the solution, along with all the DAX code that needs to be
implemented in order to solve the scenario. We kept the
description of the code short, using comments in the code to
document the measures where needed.
 You need separate companion content for the book. At the
beginning of each chapter, a short URL points to the
corresponding pattern on the daxpatterns.com website. You can
download the sample files for Power BI and Excel from the
website.
  The book is intended to be used as a reference. When you want
to implement a pattern, you do not want to read long descriptions:
you want to see the code and the reason for it. Therefore, we kept
it as compact as possible, keeping the spotlight on the DAX code.
 That said, if you want to implement a pattern we strongly
suggest that you read the entire chapter before implementing any
code. The reason is that we sometimes present multiple solutions
and you need to choose the best for your specific scenario. For
each pattern we also provide the demo files both in Power BI and
Power Pivot for Excel. Sometimes the code of the two versions is
slightly different. The book always presents the Power BI solution,
which is using the latest features of DAX at the time of printing.
Some of those features are not available in Power Pivot – like
calculated tables. This is the main reason for the differences.
  There is only one exception: time-related calculations. As we
said, we gave the time-related calculations more space in the
book: we now present four different patterns for time-related
calculations. Each of these four patterns is huge. Together, they
represent more than 40% of this book. This is why we created an
introductory chapter to the time-related calculations, which aims to
help you choose the right pattern for your scenario. If you need to
implement time-related calculations, make sure to read the
introduction first, and then the full chapter covering the pattern you
decide to use.

Prerequisites
One word of advice to our readers: this book does not teach DAX.
  You are expected to already know DAX to make the best use of
these patterns. Most of the patterns show advanced DAX
techniques that you are welcome to study and use in your
solutions. By reading this book you will not learn DAX. But if you
already know DAX, you will likely become a better DAX developer.
 We suggest that you use these patterns with the latest version of
Power BI or Excel, because DAX evolves and improves over time.
We tested the patterns on Power BI June 2020, Excel 2019, and
Excel for Microsoft 365 version 2006. Most of the patterns work
with earlier versions of Power BI and Excel, but we cannot
guarantee this because we did not thoroughly test for all the
previous versions.

Acknowledgments
Last, but not least: the acknowledgments section.
  The most important person we want to thank is you. This work
was made possible by the discussions we have had over time with
readers, users, customers, and students like yourself. Therefore,
even without knowing it you have contributed to this content; and
if you post comments in our public forums, you will be contributing
further.
  That said, there are some people who directly contributed to the
entire writing process: Daniil Maslyuk meticulously reviewed each
pattern, found all the errors we had made and provided invaluable
feedback. Claire Costa reviewed our English grammar and
readability, making the book more precise and enjoyable. Sergio
Murru built the Excel versions of the sample files, which made the
patterns available also to Power Pivot for Excel users. Daniele
Perilli is the reason behind the book and the website being as
beautiful as they are. We are responsible for the content and for
any mistake, but if you can read accurate numbers, in good
English, in both Excel and Power BI, and with a gorgeous overall
presentation, it is thanks to them.
 Enjoy DAX!
