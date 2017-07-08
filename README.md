# swigged.cuda.8.0

This project is a [SWIG](http://swig.org)-generated wrapper for the CUDA Driver API version 8.0 in C#, compiled
under Net Standard 1.1.

Virtually all of the API is exposed. However, the interface with the Driver API is minimal.
For example, arrays must be marshaled into a pinned array, and an address of the array converted into an
System.IntPtr for use with the wrapper API.

Also, the purpose of this wrapper API is for support of [Campy](http://campynet.com/), an
extension to C# for GP-GPU programming that I am writing. Consequently, be forewarned that
I use only a subset of the Driver API, and much of the API is untested. (At some point, this will change.)

This wrapper is Net Standard 1.1 compliant. Therefore, it is can be used in almost any Net Framework,
Net Standard, or Net Core app or library. However, it has not been ported to Linux, and it is untested under
Mono.

# Targets

* Windows 10 (x86 and x64)

## Using the API from NuGet

#### Net Framework App on Windows

Use the Package Manager GUI in VS 2017 to add in the package "swigged.cuda". Or,
download the package from NuGet (https://www.nuget.org/packages/swigged.cuda) and
add the package "swigged.cuda" from the nuget package manager console.

Set up the build of your C# application with Platform = "AnyCPU", Configuration = "Debug" or "Release". In the Properties for the
application, either un-check "Prefer 32-bit" if you want to run as 64-bit app, or checked if you want to run as a 32-bit app.

*You will need to copy swigged.cuda.native.dll to the executable directory.*

Note, CUDA 8.0 development normally requires VS 2015. However, there is no such restriction when using Swigged.cuda.
However, if you intend to build a private copy of the wrapper API, you will need VS 2015 to compile the native libary
of Swigged.cuda.

# Example #

~~~~
using System;
using System.Runtime.InteropServices;
using Swigged.Cuda;

namespace ConsoleApp1
{
    class Program
    {
        static unsafe void Main(string[] args)
        {
            Cuda.cuInit(0);

            // Device api.
            var res = Cuda.cuDeviceGet(out int device, 0);
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();
            res = Cuda.cuDeviceGetPCIBusId(out string pciBusId, 100, device);
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();
            res = Cuda.cuDeviceGetName(out string name, 100, device);
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();

            res = Cuda.cuCtxCreate_v2(out CUcontext cuContext, 0, device);
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();
            string cu_kernel = @"
#include <stdio.h>

__global__
void kern(int * ar)
{
	int i = threadIdx.x;
	if (i < 11)
		ar[i] = ar[i] + 1;
}
";
            string compile_string = @"
nvcc --ptx --gpu-architecture=sm_20 -ccbin ""C:\Program Files(x86)\Microsoft Visual Studio 14.0\VC\bin"" y.cu";

            string kernel = @"
//
// Generated by NVIDIA NVVM Compiler
//
// Compiler Build ID: CL-21373419
// Cuda compilation tools, release 8.0, V8.0.55
// Based on LLVM 3.4svn
//

.version 5.0
.target sm_20
.address_size 64

	// .globl	_Z4kernPi

.visible .entry _Z4kernPi(
	.param .u64 _Z4kernPi_param_0
)
{
	.reg .pred 	%p<2>;
	.reg .b32 	%r<4>;
	.reg .b64 	%rd<5>;


	ld.param.u64 	%rd1, [_Z4kernPi_param_0];
	mov.u32 	%r1, %tid.x;
	setp.gt.s32	%p1, %r1, 10;
	@%p1 bra 	BB0_2;

	cvta.to.global.u64 	%rd2, %rd1;
	mul.wide.s32 	%rd3, %r1, 4;
	add.s64 	%rd4, %rd2, %rd3;
	ld.global.u32 	%r2, [%rd4];
	add.s32 	%r3, %r2, 1;
	st.global.u32 	[%rd4], %r3;

BB0_2:
	ret;
}
";
            IntPtr ptr = Marshal.StringToHGlobalAnsi(kernel);
            res = Cuda.cuModuleLoadData(out CUmodule cuModule, ptr);
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();
            res = Cuda.cuModuleGetFunction(out CUfunction helloWorld, cuModule, "_Z4kernPi");
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();
            int[] v = { 'G', 'd', 'k', 'k', 'n', (char)31, 'v', 'n', 'q', 'k', 'c' };
            GCHandle handle = GCHandle.Alloc(v, GCHandleType.Pinned);
            IntPtr pointer = IntPtr.Zero;
            pointer = handle.AddrOfPinnedObject();
            res = Cuda.cuMemAlloc_v2(out IntPtr dptr, 11*sizeof(int));
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();
            res = Cuda.cuMemcpyHtoD_v2(dptr, pointer, 11*sizeof(int));
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();

            IntPtr[] x = new IntPtr[] { dptr };
            GCHandle handle2 = GCHandle.Alloc(x, GCHandleType.Pinned);
            IntPtr pointer2 = IntPtr.Zero;
            pointer2 = handle2.AddrOfPinnedObject();

            IntPtr[] kp = new IntPtr[] { pointer2 };
            fixed (IntPtr* kernelParams = kp)
            {
                res = Cuda.cuLaunchKernel(helloWorld,
                    1, 1, 1, // grid has one block.
                    11, 1, 1, // block has 11 threads.
                    0, // no shared memory
                    default(CUstream),
                    (IntPtr)kernelParams,
                    (IntPtr)IntPtr.Zero
                );
            }
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();
            res = Cuda.cuMemcpyDtoH_v2(pointer, dptr, 11*sizeof(int));
            if (res != CUresult.CUDA_SUCCESS) throw new Exception();
            Cuda.cuCtxDestroy_v2(cuContext);
        }
    }
}
~~~~

## Alternative CUDA Driver APIs for C#

##### ManagedCuda 8.0 https://www.nuget.org/packages/ManagedCuda-80/  http://kunzmi.github.io/managedCuda/


"ManagedCuda aims an easy integration of NVidia's CUDA in .net applications written in C#, Visual Basic or any other .net language."
Although it is very good, it just isn't compatible with Net Standard and Net Core apps and libraries.

##### CSCuda https://www.nuget.org/packages/CSCuda/  https://github.com/DNRY/CSCuda

I haven't tried this, but it looks like a fine wrapper library, albeit it does not seem to expose
the CUDA Driver API, rather the CUDA Runtime API, NPP, and CUBLAS.

##### Other APIs

ILGPU does contains a wrapper library for the CUDA Driver API, but it is
not a stand-alone API, and it is only a small subset sufficient for compilation
of C# into PTX.
