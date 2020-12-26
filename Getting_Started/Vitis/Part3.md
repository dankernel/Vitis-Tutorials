# Vitis Flow 101 – 파트 3 : 벡터 추가 예제 만나기

 이 자습서에 사용 된 예제는 간단한 벡터 추가 응용 프로그램입니다. 이 예제의 단순성은 복잡한 알고리즘 고려 사항에주의를 분산시키지 않고 FPGA 가속의 핵심 개념에 집중할 수 있도록합니다.

 

## 벡터 추가 커널의 소스 코드

이 튜토리얼에서 하드웨어 가속기 (커널이라고도 함)는 C ++로 모델링됩니다. Vitis 흐름은 Verilog 또는 VHDL로 코딩 된 커널도 지원합니다. 벡터 추가 커널의 Verilog RTL 버전을 사용하는 예는 [여기](https://github.com/Xilinx/Vitis_Accel_Examples/tree/master/rtl_kernels/rtl_vadd)에서 찾을 수 있습니다.

C ++를 사용하면 하드웨어 가속기에 대한 설명이 20 줄 미만의 코드에 적합하며 Vitis 컴파일러를 사용하여 FPGA에서 쉽고 효율적으로 구현할 수 있습니다.

```cpp
extern "C" {
  void vadd(
    const unsigned int *in1, // Read-Only Vector 1
    const unsigned int *in2, // Read-Only Vector 2
    unsigned int *out,       // Output Result
    int size                 // Size in integer
  )
  {

#pragma HLS INTERFACE m_axi port=in1 bundle=aximm1
#pragma HLS INTERFACE m_axi port=in2 bundle=aximm2
#pragma HLS INTERFACE m_axi port=out bundle=aximm1

    for(int i = 0; i < size; ++i)
    {
      out[i] = in1[i] + in2[i];
    }
  }
}
```


이 간단한 예제는 C ++ 커널의 두 가지 중요한 측면을 강조합니다.
1. Vitis는 이름 맹글링 문제를 피하기 위해 C ++ 커널을`extern "C"`로 선언해야합니다.
2. Vitis 컴파일 프로세스의 결과는 소스 코드에서 pragma를 사용하여 제어됩니다.

이 외에 벡터 추가 커널의 기능은 매우 쉽게 인식 할 수 있습니다. vadd 함수는 두 개의 입력 벡터 (in1 및 in2)를 읽고 간단한 for 루프를 사용하여 출력 벡터에 추가합니다. 'size'매개 변수는 입력 및 출력 벡터의 요소 수를 나타냅니다. 

pragma는 함수 매개 변수를 별개의 커널 포트에 매핑하는 데 사용됩니다. 두 입력 매개 변수를 다른 입력 포트에 매핑하면 커널이 두 입력을 병렬로 읽을 수 있습니다. 일반적으로이 입문 자습서에서 더 자세한 내용을 다루지 않고 하드웨어 가속기의 인터페이스 요구 사항을 고려하는 것이 중요하며 달성 가능한 최대 성능에 결정적인 영향을 미칩니다.

Vitis 온라인 문서는 [C ++ 커널 코딩 고려 사항](https://www.xilinx.com/html_docs/xilinx2020_2/vitis_doc/devckernels.html#rjk1519742919747)에 대한 포괄적 인 정보와 전체 [pragma 참조 가이드](https://www.xilinx.com/html_docs/xilinx2020_2/vitis_doc/tfo1593136615570.html)를 제공합니다.




## 호스트 프로그램의 소스 코드

호스트 프로그램의 소스 코드는 C / C ++로 작성되었으며 표준 OpenCL API를 사용하여 하드웨어 가속 벡터 추가 커널과 상호 작용합니다.

* 이 튜토리얼의 `src` 디렉토리에 있는 [`host.cpp`](./example/src/host.cpp) 파일을 엽니다.

이 간단한 예제의 소스 코드에는 4 가지 주요 단계가 있습니다.

* 1 단계 : OpenCL 환경이 초기화됩니다. 이 섹션에서 호스트는 연결된 Xilinx 장치를 감지하고 파일에서 FPGA 바이너리 (.xclbin 파일)를로드 한 다음 찾은 첫 번째 Xilinx 장치로 프로그래밍합니다. 그런 다음 명령 대기열과 커널 개체가 생성됩니다. 모든 Vitis 애플리케이션에는이 섹션의 코드와 매우 유사한 코드가 있습니다.

* 2 단계 : 애플리케이션은 커널과 데이터를 공유하는 데 필요한 세 개의 버퍼를 생성합니다. 각 입력에 대해 하나씩, 출력에 대해 하나씩. 데이터 센터 플랫폼에서는 4k 페이지 경계에 정렬 된 메모리를 할당하는 것이 더 효율적입니다. 임베디드 플랫폼에서는 연속적인 메모리 할당을 수행하는 것이 더 효율적입니다. 이들 중 하나를 달성하는 간단한 방법은 버퍼를 생성 할 때 Xilinx 런타임이 호스트 메모리를 할당하도록하는 것입니다. 이는 버퍼를 생성 할 때`CL_MEM_ALLOC_HOST_PTR` 플래그를 사용하고 할당 된 메모리를 사용자 공간 포인터에 매핑함으로써 수행됩니다.

```cpp
    // Create the buffers and allocate memory   
    cl::Buffer in1_buf(context, CL_MEM_ALLOC_HOST_PTR | CL_MEM_READ_ONLY,  sizeof(int) * DATA_SIZE, NULL, &err);

    // Map host-side buffer memory to user-space pointers
    int *in1 = (int *)q.enqueueMapBuffer(in1_buf, CL_TRUE, CL_MAP_WRITE, 0, sizeof(int) * DATA_SIZE);
```

*참고 : 일반적인 대안은 응용 프로그램이 명시 적으로 호스트 메모리를 할당하고 버퍼를 만들 때 해당 포인터를 재사용하는 것입니다. 이 예제에서 사용 된 접근 방식은 데이터 센터 및 임베디드 플랫폼 모두에서 가장 이식 가능하고 효율적이기 때문에 선택되었습니다.*


* 3 단계 : 호스트 프로그램은 커널의 인수를 설정 한 다음, 두 개의 입력 벡터를 장치 메모리로 전송, 커널 실행, 마지막으로 결과를 호스트 메모리로 다시 전송하는 세 가지 작업을 예약합니다. 이러한 작업은 1 단계에서 선언 된 명령 대기열에 추가됩니다.이 세 가지 함수 호출은 비 차단이라는 점을 기억하는 것이 중요합니다. 명령은 대기열에 배치되고 Xilinx 런타임은 명령을 장치에 제출합니다. 이 예제에서 호스트 코드에 사용 된 큐는 순서가 지정된 큐이므로 이러한 명령은 지정된 순서로 실행됩니다. 그러나 큐는 비 차단 호출이 순서가 아닌 준비 될 때 실행되는 비 순차적 큐일 수도 있습니다. 대기열에 포함 된 모든 명령이 완료 될 때까지 기다리려면`q.finish ()`를 호출해야합니다. 

```cpp
    // Set kernel arguments
    krnl_vector_add.setArg(0, in1_buf);
    krnl_vector_add.setArg(1, in2_buf);
    krnl_vector_add.setArg(2, out_buf);
    krnl_vector_add.setArg(3, DATA_SIZE);

    // Schedule transfer of inputs to device memory, execution of kernel, and transfer of outputs back to host memory
    q.enqueueMigrateMemObjects({in1_buf, in2_buf}, 0 /* 0 means from host*/); 
    q.enqueueTask(krnl_vector_add);
    q.enqueueMigrateMemObjects({out_buf}, CL_MIGRATE_MEM_OBJECT_HOST);

    // Wait for all scheduled operations to finish
    q.finish();
```

* 4 단계 : 이전에 대기열에 넣은 모든 작업이 완료되면`q.finish ()`호출이 반환됩니다. 이 경우 커널의 결과를 포함하는 출력 버퍼가 호스트 메모리로 다시 마이그레이션되었으며 소프트웨어 응용 프로그램에서 안전하게 사용할 수 있음을 의미합니다. 여기서는 프로그램이 완료되기 전에 예상 값과 비교하여 결과를 간단히 확인합니다.


이 예제는 OpenCL API를 사용하여 하드웨어 가속기와 상호 작용하는 가장 간단한 방법을 보여줍니다. 항상 그렇듯이 추가 정보는 [Vitis 문서](https://www.xilinx.com/html_docs/xilinx2020_1/vitis_doc/devhostapp.html#vpy1519742402284) 에서 찾을 수 있습니다.



## 다음 단계

이제 커널과 호스트 프로그램의 소스 코드를 이해 했으므로, [**4 단계**](./Part4.md)에서 이 예제를 컴파일하고 실행하는 방법을 설명합니다.

 

<p align="center"><sup>Copyright&copy; 2020 Xilinx</sup></p>
