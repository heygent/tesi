OUT=out

default:
	pandoc -t latex --toc --latex-engine=xelatex -H src/header.tex -B src/frontespizio.tex src/00-intro.md src/01-hadoop.md src/metadata.yaml -o $(OUT)/tesi.pdf 
tex:
	pandoc -t latex --toc --standalone --latex-engine=xelatex src/00-intro.md -o $(OUT)/tesi.tex
	
