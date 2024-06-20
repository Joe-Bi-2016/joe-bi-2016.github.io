---
title: "矩阵乘法的并行算法"
collection: ai
permalink: /ai/ai-01
excerpt: ' '
date: 2024-06-20
citation: 'Joe-Bi. (2024). &quot;矩阵乘法-并行算法.&quot; <i>GitHub Joe-Bi of Bugs</i>'
---
   
矩阵乘法的并行算法，是借助cuda或opencl异构计算平台来实现的。
cuda/opencl是cpu的协处理器的，被设计为提高计算能力的并行计算平台，与操作系统的线程不同，cuda/opencl是从物理层面真正支持成千上万的多线程的。

下面是矩阵乘法的实现代码，详情看注释

A(M x p) X B(p x N) = C(M x N)  

<font color="red" size=5>注意:</font> 
<br  />  
A matrix：M行p列  
B matrix：p行N列  
C matrix：M行N列  

<font color="red" size=5>矩阵全局索引计算:</font>   
<br  />
A矩阵全局索引计算：gRow范围与M一致，gRow*p是在A中的每行起始地址，k的范围是[TS, 2TS, ..., (p/TS) * TS],所以是块的行大小倍数，lCol是每块中行的索引，故对A矩阵，全局索引AgIdx = gRow * p + k + lCol;  

B矩阵全局索引计算：每块数据的行大小也是k,B矩阵的全局索引行号为k + lRow, 每行大小为N，gCol范围与N一致，故对B矩阵，全局索引BgIdx = (k + lRow) * N + gCol;  

### cuda实现

```
template<unsigned int TS>
__global__ void matrix_mul(int* A, int* B, int* C, int M, int N, int P)
{
	int lRow = threadIdx.y;	// local row index
	int lCol = threadIdx.x;	// local col index
	int gRow = blockIdx.y * blockDim.y + threadIdx.y; // global row index
	int gCol = blockIdx.x * blockDim.x + threadIdx.x; // global col index
	
	__shared__ int Asub[TS][TS];
	__shared__ int Bsub[TS][TS];

	int acc = 0;
	
	for(int k = 0; k < P; k += TS)
	{
        Asub[lRow][lCol] = A[gRow * P + lCol + k];
        Bsub[lRow][lCol] = B[(lRow + k) * N + gCol];

		__syncthreads();

		for(int k = 0; k < TS; ++k)
			acc += Asub[lRow][k] * Bsub[k][lCol];

		// Synchronise before loading the next tile
		__syncthreads();
	}

	printf("C[%d][%d] = %d", gRow, gCol, acc);

	C[gRow * N + gCol] = acc;
}
```

### opencl实现

```
#define TS 16
__kernel void matrix_mul(const __global int* A, const __global int* B, __global int* C, int M, int N, int P)
{
	const int lRow = get_local_id(1);	// local row id
	const int lCol = get_local_id(0);	// local col id
	const int gRow = get_global_id(1);	// global row id
	const int gCol = get_global_id(0);	// global col id

	__local int Asub[TS][TS];
	__local int Bsub[TS][TS];

	int acc = 0;
	
	for(int k = 0; k < P; k += TS)
	{
        Asub[lRow][lCol] = A[gRow * P + lCol + k];
        Bsub[lRow][lCol] = B[(lRow + k) * N + gCol];

		barrier(CLK_LOCAL_MEM_FENCE);

		for(int k = 0; k < TS; ++k)
			acc += Asub[lRow][k] * Bsub[k][lCol];

		// Synchronise before loading the next tile
		barrier(CLK_LOCAL_MEM_FENCE);
	}

	printf("C[%d][%d] = %d", gRow, gCol, acc);

	C[gRow * N + gCol] = acc;
}
```

cuda的更进步优化手段还可以结合配置文件优化线程参数、nvprof工具查看效率时效比、全局内存模型的内存事务合并对齐访问，一级缓存/共享内存与全局内存颗粒度的差异、内存带宽传输比等等手段。

opencl的优化可借助一些数据内存存储特点和内置函数合并访问提高效率等手段。
