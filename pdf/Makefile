SRCDIR = src
OUTDIR = out
SRC = $(sort $(wildcard $(SRCDIR)/*.md))
HEADER = $(SRCDIR)/header.tex
TITLEPAGE = $(SRCDIR)/frontespizio.tex
METADATA = $(SRCDIR)/metadata.yaml
BIBLIOGRAPHY = $(SRCDIR)/references.yaml

FLAGS = -t latex --toc --latex-engine=xelatex -F pandoc-crossref -F pandoc-citeproc
FLAGS += --top-level-division=chapter

$(OUTDIR)/tesi.pdf $(OUTDIR)/tesi.tex: $(SRC) $(HEADER) $(TITLEPAGE) $(METADATA) $(BIBLIOGRAPHY)
	pandoc $(FLAGS) -H $(HEADER) -B $(TITLEPAGE) $(METADATA) $(SRC) -o $@ --csl=springer-basic-brackets-no-et-al-alphabetical.csl
	

