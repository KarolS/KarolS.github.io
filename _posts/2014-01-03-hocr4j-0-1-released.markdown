---
layout: post
title: "hOCR4J 0.1 released"
date: 2014-01-03 00:11
categories: [programming, ocr, java, "personal-project", "hocr4j"]
---

I would like to announce the release of a Java library for parsing hOCR documents: **hOCR4J**. You can [download it from here](https://github.com/KarolS/hOCR4J). I'm planning to get it to Sonatype too, so you may be able to get it from there in the near future.

hOCR is an output format used by OCR programs, including [Tesseract](http://code.google.com/p/tesseract-ocr). It contains information about all the OCR'd words, their position, and their assumed organisation into lines and paragraphs. Currently, hOCR4J was tested to work with Tesseract-generated hOCR's, I plan to test other OCR programs in the future.

hOCR4J parses hOCR documents, creates an immutable model for them (nice when using functional programming style), and provides various tools to manipulate and modify them.

hOCR4J makes a good starting point when developing an application which extracts data from OCR'd documents that have non-trivial layouts.

<!-- more -->

The model of an hOCR document is simple: a page contains areas, an area contains paragraphs, a paragraph contains lines, a line contains words. Each of these objects has a bounding box, which defines its position on the scanned page. hOCR4J provides various operations on bounding boxes in the `Bounds` class, including scaling, resizing, translating, unions, intersections, and more.

Using hOCR4J is also simple. First, we need to get our hOCR file and read the hOCR into a string. Then we can parse it:

```java
String hocr = ... ; // load hOCR here

List<Page> pages = HocrParser.parse(hocr);

Page page0 = pages.get(0);
```

We can now extract some text:

```java
List<String> textLines = page0.getAllLinesAsStrings();
```

We can only extract lines that satisfy some conditions:

```java
List<Line> lines = page0.findAllLines(LineThat.matchesRegex("^IMPORTANT:"));

for (Line line: lines){
  String text = line.mkString();
  // ...
}
```

We can look for words in italics:

```java
List<Word> words = page0.getAllWords();

for (Word word: words){
  if (word.isItalic()){
    String w = word.getText();
    // ...
  }
}
```

We can look for location of a word or phrase (spaces are ignored, as OCR sometimes inserts more or less of them):

```java
// let's censor the name of the culprit
Page censoredPage = page0.mapLines(new Function<Line,Line>(){
  public Line apply(Line line){

    // we check if the line mentions culprit's name
    final Bounds theCulpritIs = line.findBoundsOfWord("The culprit is");

    if (theCulpritIs != null) {
      // we take bounds of the entire line
      Bounds full = line.getBounds();

      // and we calculate bounds of a line
      // that doesn't extend beyond words "The culprit is"
      Bounds remain = new Bounds(
        full.getLeft(), 
        full.getTop(), 
        theCulpritIs.getRight(), 
        full.getBottom());

      // finally, we trim our line
      return line.createBounded(remain);
    } else {
      // all the other lines are left unmodified
      return line;
    }
  }
});

return censoredPage;
```

In the near future, I'm planning a tutorial on extracting text columns from hOCR using hOCR4J.