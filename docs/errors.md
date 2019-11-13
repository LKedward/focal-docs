# Error handling

An error handler is included automatically and is called after every underlying OpenCL API call to check for errors.
If an OpenCL call does not return CL_SUCCESS, then the default error handler will indicate where this occurred and will halt the program.

The following [blog post](https://streamhpc.com/blog/2013-04-28/opencl-error-codes/) on streamhpc.com provides a comprehensive description of possible OpenCl error codes and is useful for debugging.



## 1. Custom error handling

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
