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
For example, in gfortran this is done with the `-I` flag:

```shell
$> gfortran -c myprogram.f90 -I/path/to/focal/mod/ -o myprogram.o
```

## Link
To link, you need to specify the path to the Focal repository and directives for `Focal` and `OpenCL`:

```shell
$> gfortran myprogram.o -L/path/to/focal/lib/ -lfocal -lOpenCL -o bin/myprogram
```

If the compiler cannot find `-lOpenCL` then you will additionally need to point the compiler to the relevant OpenCL SDK library file (`libOpenCL.dll/.so`):

```shell
$> gfortran myprogram.f90 -L/path/to/focal/lib/ -lfocal -L/path/to/OpenCL/lib/ -lOpenCL -o myprogram
```

See examples in the repository for how this can be done with a `makefile`.


## Debug build
When the Focal library is built, two versions are produced: `libFocal` and `libFocaldbg` where the latter
contains additional runtime calls that check the validity of your program.

To link against the debug build replace `-lFocal` with `-lFocaldbg` in the linking command.
See [here](../errors#2-runtime-debug-checks) for more information on runtime debug checks.
