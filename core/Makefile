.PHONY: core clean

core:
	mkdir -p build
	cd build && cmake ../ -DUSE_CUDA=ON && make $(MAKEFLAGS)

clean:
	rm -rf build/
