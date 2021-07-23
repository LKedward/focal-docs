# Errors

There are three classes of errors to consider when writing an OpenCL application with Focal:

- __Compile-time:__ errors which occur when compiling a program and which prevent a valid
program from being created successfully. A compile-time error relating to a Focal library call
likely indicates that you are using incompatible types (*e.g. transferring a float to an integer*).

- __OpenCL runtime:__ errors which occur during an OpenCL API call. All API calls are checked by
a customisable error handler.

- __Focal runtime:__ errors which occur in the Focal library. These can include optional validity checks
that are enabled when linking against the [debug build](../linking#debug-build).

The remainder of this guide relates to defining a custom error handler for OpenCL API errors and using the additional
runtime checks provided in the debug build.

## 1. OpenCL error handling
An OpenCL error handler is included automatically and is called after every underlying OpenCL API call to check for errors.
If an OpenCL call does not return CL_SUCCESS, then the default error handler will indicate where this occurred and will halt the program.
The following [blog post](https://streamhpc.com/blog/2013-04-28/opencl-error-codes/) on streamhpc.com provides a comprehensive description of possible OpenCl error codes and is useful for debugging.

If you would like to handle OpenCL errors yourself, then you can specify your own error handler.
Custom error handlers must conform to the following subroutine interface:

```fortran
abstract interface
    subroutine fclErrorHandlerInterface(errcode,focalCall,oclCall)
      use iso_c_binding
      integer(c_int32_t), intent(in) :: errcode
      character(*), intent(in) :: focalCall
      character(*), intent(in) :: oclCall
    end subroutine fclErrorHandlerInterface
end interface
```

The first argument `errcode` is the integer error code returned by the OpenCL API call.
A utility is provided in Focal to lookup the corresponding error string `fclGetErrorString(errcode)`.

The second argument is a character string indicating in which Focal API call the error code was produced.

The third argument is a character string indicating in which OpenCL API call the error code was produced.

Having written an error handling subroutine `myErrorHandler` conforming to the above interface, you can enact it using the global pointer `fclErrorHandler` at the beginning of your program:

```fortran
fclErrorHandler => myErrorHandler
```

As an example, the Focal default error handler is defined by:

```fortran
subroutine fclDefaultErrorHandler(errcode,focalCall,oclCall)
    use iso_c_binding
    integer(c_int32_t), intent(in) :: errcode
    character(*), intent(in) :: focalCall
    character(*), intent(in) :: oclCall

    if (errcode /= CL_SUCCESS) then

      write(*,*) '(!) Fatal openCl error ',errcode,' : ',trim(fclGetErrorString(errcode))
      write(*,*) '      at ',focalCall,':',oclCall

      stop 1
    end if

end subroutine fclDefaultErrorHandler
```

__API ref:__
[fclGetErrorString](https://lkedward.github.io/focal/interface/fclgeterrorstring.html),
[fclHandleErrorInterface](https://lkedward.github.io/focal/interface/fclhandleerrorinterface.html),
[fclDefaultErrorHandler](https://lkedward.github.io/focal/interface/fcldefaulterrorhandler.html),
[fclErrorHandler](https://lkedward.github.io/focal/module/focal.html#variable-fclerrorhandler)

## 2. Runtime debug checks

To enable runtime debug checks, replace the link flag `-lFocal` with `-lFocaldbg` when [linking](../linking) your program.

The following runtime checks are performed when using the debug build:

- __Number of kernel arguments:__ checks that the number of arguments passed at runtime
 matches the actual number of arguments the OpenCL kernel accepts.

- __Type of kernel arguments:__ checks that the types of arguments passed at runtime
match the types that the OpenCL kernel is expecting. This includes argument address spaces.

- __Device buffer initialisation:__ checks that device buffer objects are initialised
when performing memory operations.

- __Size of device buffers:__ checks that the size of device buffers is valid when
performing memory operations such as write, read, and copy.

- __Kernel event execution status:__ waits after each kernel launch and checks for error status.
This is important for ensuring kernels are executing correctly.
If an error code is detected, it is printed before aborting the program.

!!! note
    Since each kernel launch is wait upon and status-checked in the debug build, then kernel launches
    become __blocking__ operations. Kernel launches are unaffected (non-blocking) in the normal build.

When a runtime error is detected the program is aborted immediately such that a stack trace is printed,
and which can be used to detect the source of the error.

__API ref:__
[fclDbgWait](https://lkedward.github.io/focal/interface/fcldbgwait.html)
