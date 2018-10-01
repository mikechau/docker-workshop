.PHONY: build clean

build: clean
	GO111MODULE=on CGO_ENABLED=0 go build -a -installsuffix cgo -o dist/app

clean:
	rm -rf dist
