# Linking Focal Programs
 
## Include
To use the Focal module in your own code, include the module with the `use` syntax:

```fortran
program myprogram
use Focal
implicit none
...
end program myprogram
```

## Compile
To compile a program using the Focal module, you must point the compiler to the `mod/Focal.mod` definition in the Focal repository.
For example, in gfortran this is done with the `-J` flag:

```sh
gfortran -c myprogram.f90 -J/path/to/focal/mod/ -o myprogram.o
```

## Link
To link, you need to specify the path to the Focal repository and directives for `focal` and `OpenCL`:

```sh
gfortran myprogram.o -L/path/to/focal/lib/ -lfocal -lOpenCL -o bin/myprogram
```

If the compiler cannot find `-lOpenCL` then you will additionally need to point the compiler to the relevant OpenCL SDK library file (`libOpenCL.dll/.so`):

```sh
gfortran myprogram.f90 -L/path/to/focal/lib/ -lfocal -L/path/to/OpenCL/lib/ -lOpenCL -o myprogram
```

See examples in the repository for how this can be done with a `makefile`.
