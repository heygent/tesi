SRCDIR = src
OUTDIR = out
SRC = $(sort $(wildcard $(SRCDIR)/*.md))
HEADER = $(SRCDIR)/header.tex
TITLEPAGE = $(SRCDIR)/frontespizio.tex
METADATA = $(SRCDIR)/metadata.yaml
BIBLIOGRAPHY = $(SRCDIR)/references.yaml

FLAGS = -t latex --toc --latex-engine=xelatex -F pandoc-citeproc

$(OUTDIR)/tesi.pdf $(OUTDIR)/tesi.tex: $(SRC) $(HEADER) $(TITLEPAGE) $(METADATA) $(BIBLIOGRAPHY)
	pandoc $(FLAGS) -H $(HEADER) -B $(TITLEPAGE) $(SRC) $(METADATA) -o $@ --csl=ferrara.csl
	
