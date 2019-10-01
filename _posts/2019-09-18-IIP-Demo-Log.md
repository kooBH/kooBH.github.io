---
layout : default
title:  "IIP Demo Log"
date:   2019-09-18 17:30:00
---

### 01.07  
#### TODO
1. RTaudio 8ch 16,000 input
2. Wav file input as Real-Time
3. How Blaze works  

BLAZE-lib 에서 Cmake가 하는 일 
  1. 기기의 아키텍쳐를 분석해서 적합한 cache size를 구현 - CPU의 캐시 크기에 맞게 연산 단위를 조정하는 것이 아닐까
  2. find_package()와 그에 따른 set_definition - 빌드 환경에 따라 분기하는 전처리 구문이 존재하며 이는 cmake를 통해 설정된다.  

Q. 실시간 인풋과 데모 인풋 관련된 변수 관리  
  -  같이 묶을 까? 따로 쓸까? => 따로 쓰자

Q. 실시간 연산 어떻게 구현하나?
  - RtAudio를 128프레임받고 연산 받고 연산? 1초에 16000개 받는 데도?
  - 길게 받고 버퍼에서 조금씩 받아서 실시간 연산? => 조금 받을 때 마다 그거 처리하라고 다음 단계를 콜백해줘야 한다

### 01.08

+ __LINUX__ALSA__ 가 정의되어 있지않다. RtApiDummy만 생성함 -> cmake settting 필요했음
+ 입력스트림의 분리 - 별도의 쓰레드
+ 오디오 스트림이 계속 돌아가는 동안 각 프레임 단위에 대해 프로세스를 실시간으로 해줘야한다
+ 일단 웨이브 파일 관리먼저 하자 - short type 입력만 받는가? -> short 만 받음
+ 웨이브 파일 관리를 어떻게 할까? - 기능의 분할과 파일 관리
+ WAV는 어느정도 끝냄. premature eof 가 걸리긴하는데, RTaudio와의 콜백 타이밍 문제를 해결하고 봐야할듯

#### TODO
1. ShiftFrame 단위로 RT_input에서 받아서 TotalFrame으로 overlap해서 process block으로 보내기
2. 보내는 형태와 방식에 대해 생각해보자

### 01.09
RT_Input에 stock이라는 변수를 넣어서 처리해야하는 데이터가 있는지 확인후 그것을 가져와서 연산하도록함.  
stock > max_stock 시 Fail  
FFT 추가함  
처리 후 output overlap 함-> 잘되는지 확인하는 방법을 생각해보자.  
WAV duration 이 잘못 Write 되어있다. 고침 -> data_size 과하게 설정했었다  
OpenMP 추가하자. - CMake추가함 -> 아직까진 할만큼 연산량이 많은 부분이 없다.  
Blaze를 알아보자 -> CMake시 Blaze의 install을 같이 할 수 있게 해보자. -> Blaze의 CMakeLists.txt 도 별도의 Target을 가지고 있다.   이 문제와 경로문제를 해결하면 include로 가능 할 지도?  
-> 일단 Default 값들로도 most 장치에서 좋은 성능을 보인다하니 CMake를 합치는것은 추후에..  
-> 현재 기본 Blaze.h를 사용하도록 해놨다 -> 빌드시간이 많이 늘어남   

#### TODO
입출력을 Blaze의 자료형과 호환하는 방법.    
Blaze의 최적화 방식 - 좀 더 알아보기  

### 01.10
Windows func 안했다. HannWindow클래스를 만들었다.
데모용 입력을 위한 WAV 파일을 입력으로 받는 기능을 WAV class에 추가하자
HannWindow + Overlap 하면 반향이 제거된 wav가 나와야 하지만 wav가 깨져서 나온다. 뭐가 문제인가
-> 자료구조 잘 못 설계함
-> 전부다 [CHANNEL][FRAME_SIZE]로 변경함

.|.
---|---
main|o
WAV|o
RT|o
HANN|o
FFT|o
AFTER|o

-> 입력도 우측에서 부터 들어와서 shift 하는걸로 수정함

### 01.11
#### TODO
+ WPE - 함
+ micro sec 단위 타이머 - 함
+ 사용 편의성 향상

현재 fs = 48000 frame_size = 512 shift_size = 128 일 때 5 채널까지 실시간 가능  
현재 fs = 48000 frame_size = 512 shift_size = 256 일 때 7 채널까지 실시간 가능  

### 01.14
hann 이슈는 raw 와 hanned_Raw를 따로 두지 않아 hann이 중복되서 발생한 문제 -> 해결 
WPE를 얹었으나 잘 돌아가는 건지 아직 확인하지 못함. & 성능향상이 요구된다 - 실시간 8채널.. 힘들것같음

현재 fs = 48000 frame_size = 512 shift_size = 128 일 때 3 채널까지 실시간 가능 -> 코드 수정하면서 연산이 많이 추가됨     

#### TODO
작업 기기 - 8 프로세서  && 현재 545% 정도 CPU 사용률 -> 700%까지는 올려야함  
WPE가 멀티 쓰레딩 안하고 있었다. CPU 사용률 -> 683%
780%까지 씀 -> 이정도 돌아가면 더 이상 못 올라가기에 위험한 상태
저번에 WPE만 최적화 했을 때는 sample_rate가 16000 이었으나 지금은 48000으로 하고 있다  
연산량이 3배..

+ 메모리 복사 관련 게시물  
https://stackoverflow.com/questions/4707012/is-it-better-to-use-stdmemcpy-or-stdcopy-in-terms-to-performance/9980859#9980859
https://stackoverflow.com/questions/4729046/memcpy-vs-for-loop-whats-the-proper-way-to-copy-an-array-from-a-pointer


### IDEA  
각 프레임에 대한 실시간은 힘들지 않을까? 상용되는 기기들도 딜레이가 존재함  
-> VAD를 적용한다면. 되는 정도 까지는 하되 밀리는 부분은 Buffer에 담아두고 VAD가 low하게 되면 딜레이주고 그동안 나머지 버퍼를 돌리는 형식?

## TODO !!
48k to 16k Resampling.. 

### 01.15
Resampleing 추가중.. 채널 간 데이터가 침범하는 오류가 있다.    
추가적으로 가지고 있는 코드의 Shift_size = 256인데 lpf[512]이며 이중 왼쪽 절반만을 fft해서 필터링 한다..  어떻게 해야하?  
=> 일단 16k input 받기로함
-> 8ch 16k  760% 간당간당함. 기능 더 추가하면 힘들듯  
=> 위의 것은 CH = 8 이었다. 실제로는 CH = 6 + RF = 2 로 갈것
=> RF를 어디까지 가져 갈것인지는 모르겠지만 첫번째 FFT까지만 CH+RF로 가고 나머지 연산은 CH로 했을 경우  
  560% 대로 여유있는 모습을 보임  
=> 추후에 48k로 한다 해도 resampling을 하면 안정적으로 돌아갈 것으로 예상  

아.. 어떤 알고리즘은 full fft 돌리고 어떤 애들은 hfft쓰고 고정된게 없다 ㅇ벗어렁댦ㄹ무룾ㄷ라궂ㄷㄱ ㅈㄷㄱㅂㅈㄷㄱㄹ  
->그때 그때 필요하면 정리하고 추가하자 - > 알고리즘별로 fft 용법이 미미하게 차이남

VAD 추가함. 작동되는 것만 확인. 기능의 적용은 아직 하지 않음  
* VAD를 쓰려면 이전 버퍼를 어느정도 지니고 있어야함 -> 얼마나?



### 01.16
WPE 되는 것 확인함.  
현재 출력 파일의 헤더를 프로그램이 종료될 때 기록함. 프로그램 자체 예외 처리로 종료할 경우 헤더파일이 기록되나, 이외의 경우 헤더를 쓰지 않고 끝나기 때문에 출력파일을 사용할 수 없음. -> 매 기록 때 마다 파일 포인터를 저장하고 헤더를 쓰고 복귀하는 방법을 추가할까?  
-> 이게 소요시간을 유의미하게 늘리는지 확인해야함  
-> 어차피 Append는 맨 마지막에 적는 거라 포인터 관리 할 필요 없다

리샘플링은 일단 보류 - 16k 입력을 사용한다

+ VAD 기능을 적용하자.
+ NOTE) 모듈형으로 구성하려면 VAD에 대한 iuput seqeunce가 별도로 존재해야한다.

####
1. 실시간 헤더 작성 - done
2. VAD에 맞는 입력버퍼  
3. RTaudio 예외처리가 안되어있다 - > HW가 불가용일때, 없을때, 

+ VAD 들어갈 당시 FFT 를 한 값이 들어가는데, 이걸 저장해 두는게 좋겠다.
VAD는 단일 채널(1번째 채널)  

input -> data -> VAD -> epdState를 받아서 판단.   
이 data를 VAD_DELAY만큼 가지고 있어야함.  
끝 날떄도 VAD_DELAY만큼 더 해야함..

2 종류의 버퍼가 필요함  
1. VAD delay를 주기 위한 버퍼
2. VAD의 EPDstate의 유효성을 검증하는 Delay를 위한 버퍼

VAD 달았다. 잘 돌아간다. 환경이 바뀌면 튜닝만 해주면 될듯


### 01.17

#### TODO
+ 코드 정리
+ BLAZE
+ AEC
+ Interface 

출근해서 VAD 테스트 하는 데, VAD PSY값이 전부 -nan 이다. 왜지?    
=> 첫번째 채널만 테스트 하는데, 다른 채널 마이크를 향해서 소리가 발생할 경우 인식이 안되는 경우가 발생  
=> 전 채널에 대해 VAD를 시행하는 것으로 변경 -> 추후에 부하가 클 경우 조정요망   

BLAZE는 사용에 대한 공부가 필요할듯  
코드를 정리해보자.-> shift & copy를 inline으로, 


#### stereoANC 
+ step 은 process를 lambda로 감싸서 해당 lambda를 수행하는 쓰레드를 이용
+ 각 쓰레드는 channel에 대해서 수행

현재 AEC 를 위한 선언부 생성 - raw,data,fft,hann,stereoanc_rls..  
reference는 rt_input에 같이 들어옴 

#### Procedure
1. input : CH + RF  
2. raw 와 ref 로 분리
3. raw와 ref 에 각각 hann, fft
  => data,ref_data
4. VAD : IsSpeech 일 경우
5. AEC(data,ref_data,AEC_output)수행 // AEC는 inplace인가 outplace인가? 알아봐야함
6. WPE(AEC_output)수행




#### 01.18 

#### TODO

+ Output : Delay 고려한 overlap 이랑 time이 같은 원본 input 출력  -> Done
+ AEC : stereo아닌 AEC -> wav in & overlap 없는 1024 입력..(7ch -> 6ch) => 코드를 수정해야함  
  => 무식한 방식으로 구현함 작동하는 것 확인함. -> done
+ SRP(너무 지저분) & MVDR(잘 돌아가는지 모르겠음) : 코드 완성 덜됨
+ Reference input 분리
+ WPE 와 (AEC & SRP & MVDE)은 스위칭 -> 현재 별도의 main으로 구분해둠

note) stereoans_rls 는 보류  

코드들 너무 중구난방이다.. 지저분하고 아아아암ㄴㅇ랑낢ㄴ으랑ㄴ므라ㅣ

현재 VAD + monoAEC : 330% 사용
=> 종료(kill) 시 렉걸림. 

코드 정리 하자

AEC의 FFT는 https://www.netlib.org/fftpack/ 의 fft 사용함.  
http://www.kurims.kyoto-u.ac.jp/~ooura/fftbmk.html   
(Ooura 에서 제공하는 벤치마크) 에 따르면  
90년대에 개발이 끝난 netlib보단 Ooura가 빠름 (FFTW version 3 까지 나와서 논외)  
Numerical의 FFT도 사용한다
섞어서 사용한다. 정리해야할듯 => hfft와 fft 2개 별도로 존재해야함

main의 사용법 변경.. 
aec - demo는 좀 더 테스트?  
현재 WPE-demo가 에러남    



### 01.21

#### TODO

+ 기능 테스트 : gain 을 변경해서 test_input을 다시 만들어야함 - done
+ AEC - 살펴보기
+ RT_Audio 와  Wave read의 모듈식 구분

 현재 - 데모 파일에 대해선 잘 작동한다.   
sample 3 - wpe, 4 - ica 잘 되는 예제 샘플  

### 큰 문제  
환경에 따라서 적절한 값이 너무 심하게 바뀐다.  
현재 gain을 1로 하고 C#으로 맴스보드의 Db값을 32로 변경해서 돌리는데, 이러면 주변 소음이 적게 들어가는 대신, input도 작아서
VAD 가 기존의 값으로는 인식을 못해서 THr을 3.2 -> 1.5로 해서 돌려야한다. 또한 WPE postgamma 0.97은 이 입력에는 너무 낮아서 0.995로 변경하였다. 기타 환경이 바뀔경우 그에 따라 여러가지가 바뀌어야 하는 것이 많다  ..... 



### 01.24

데모 시연함. mono_aec + srp + mvdr 프로시저가 추가되었으나, 더이상 쓸것 같지는 않다고한다. 프로파일러를 좀 살펴봤으나, VS로 보는게 제일 깔끔한듯..  
stereo_AEC를 살펴보고있다. transepose를 앞뒤로 해서 SIMD연산을 가능하게 하는 것 같다. 하지만 지금은 ref를 어떻게 받아서 사용하는지 찾는 중이다.

그전에 테스트 빌드가 너무 느리고 결과가 틀리다.  
-> SIMD 적용이 안되어있었음  

```CMake
-O3
-mavx
-D__AVX2__
```
--> 에러  

내 PC의 i7-3770k에는 AVX2가 없다. native하게 연산하는건 너무 느리고 구현도 안되어있다..  



## 01.25

phoneme 서버를 사용해서 진행하려고함. 서버 접속이 안된다. permission denied만 뜸. 아이디나 비번은 경우를 다 넣어봤으나 안됨. 

.ssh 폴더를 날리고

새로 공개키를 받았으나 안됨. 

서버에서 ssh 설정을 바꿔야할거 같은데, 서버 비번도 모른다. ㅡㅡ 

그래서 AVX2 대신 AVX를 하려고 했다만
복소수 연산에 FMA를 쓰는데 그거는 현재 장비의 CPU가 지원하지 않는다.. 서피스북으로 장비를 변경하기로함 
-> Windows 호환성 개선해야함

WAV::CreateFile이 
fileapi.h 와 충돌

```C++
#define CreateFile CreateFileA
```

이라는 이상한 구문이 있다. WT..

+ Refactoring : Createfile -> NewFile

서피스북을 원격 접속으로 하려고 했는데 잘 안된다 .
 Remmina 로는 옵션 경우대로 해봤는데 안되네. 크롬 리모트도 안됨. 문제가 있다면 내 pc가 문제일듯.  


일단 그냥 git을 매개로 작업 진행중

stereoAEC::step을 in,ref 인자를 합쳐서 동작하도록함. 지금은 seg가 뜨는 상태 




### 01.28

stereoAEC test 01.. stereoAEC는 같이들어가지만 같이 나오지 않음. monoAEC는 들어가기전의 raw를 전처리해서 CH과  RF의 차를 고려하지 않아도 되었다.  

잘 작동하지 않음. test 코드의 window는 2048 shift_size 는 512였다. monoAEC처럼 128 + 128 + 128 + 128 하여 각 128마다 돌려보자.

디버깅..  .. 그냥 데이터 선형으로 두고  for문돌리면 컴파일러가 알아서 vectorizing 해줄텐데, 그냥 다 해체해버릴까?  

에러 - 20초 3채널 짜리를 12분 돌려놓고 에러남 (디버깅모드) : free_base.cpp 의 _free_base에서 에러. 항목은

```C
// This function implements the logic of free().  It is called directly by the
// free() function in the Release CRT, and it is called by the debug heap in the
// Debug CRT.
extern "C" void __cdecl _free_base(void* const block)
{
    if (block == nullptr)
    {
        return;
    }
    
    if (!HeapFree(select_heap(block), 0, block)) // <- 여기서 중단점 발생
    {
        errno = __acrt_errno_from_os_error(GetLastError());
    }
}
```

일단 해당문제는 보류 - 오래 걸릴것 같음. 

일단 linux에서 잘 돌아가는 지부터 파악하자    

ref 는 입력이 들어가는데 x가 입력이 안들어간다.  

WAV 파일은 문제가 없어 보이는데 입력받은 raw 에서
채널부분의 데이터가 

```
값 값 값 nan 값 값 값 nan 값 값 값 nan
```
이렇게 되어있다. ->  REF 부분은 잘 받음  

input wav를 새로 설정해보자


### 01.29

새로 6 + 2 로 녹음함. 
0 채널만 


```
값 값 값 nan 값 값 값 nan 값 값 값 nan
```

이렇게 되어있다.

wav_input 이상없고 raw에 들어가는 시점에도 이상이 없지만  
SHIFT를 하고나면 ch0의 4번째 값들이 다 -nan이 된다.

 찾았다

stereoAEC의 

```C++
                ch.m_D[batch] = fX ;//+ (rho * simd_float(0.5) - simd_float(1)) * GTap;
```

에서 rho 부분이 simd_float 이고 GTap이 simd_complx인데 저 부분에서 메모리 침범이 일어나는듯. 

또 다른 에러

인자로 들어온 data array를

```C++
ChannelData& ch = m_channels[channel];
            sfloat_t* rdst = reinterpret_cast<sfloat_t*>(ch.m_fX.data());
            for (size_t sample = 0; sample < m_nWin; ++sample) {
                rdst[sample] = m_hanningWin[sample] * x[channel][sample];
            }
            rfft(rdst, m_fftSize);
             for (size_t batch = 0; batch < m_nBatches; ++batch) {
                       transpose(ch.m_fX[batch]);
            }

```

이렇게 사용하는데, rdst에는 데이터가 제대로 들어가지만 rdst를 ch.m_fx의 포인터로 인식하지 않는 듯하다.
ch.m_fX에 값이 들어가 있지 않다ㅏ.

아니ㅣ

```
point 1610666048 point 1610666048 
[1]rdst -54.049896  fRef 0.000000 0.000000
[1]rdst 0.000000  fRef 0.000000 0.000000
[1]rdst -50.140345  fRef 0.000000 0.000000
[1]rdst -21.545754  fRef 0.000000 0.000000
[1]rdst -38.875182  fRef 0.000000 0.000000
[1]rdst -40.426615  fRef 0.000000 0.000000
[1]rdst -21.590686  fRef 0.000000 0.000000
[1]rdst -54.289988  fRef 0.000000 0.000000
[1]rdst -0.339692  fRef 0.000000 0.000000
[1]rdst -61.371325  fRef 0.000000 0.000000
[1]rdst 22.348360  fRef 0.000000 0.000000
[1]rdst -60.701076  fRef 0.000000 0.000000
[1]rdst 43.762976  fRef 0.000000 0.000000
[1]rdst -52.218667  fRef 0.000000 0.000000
```

```C++
 printf("point %d",memcpy(m_fRef[side].data(),rdst[m_nChannel+side],sizeof(double)*(m_nWin+2))  );
        printf(" point %d \n",m_fRef[side].data());
    for(int i=0;i<m_nBatches;i++){
            printf("[%d]rdst %lf  fRef %lf %lf\n"
                ,m_nChannel+side,rdst[m_nChannel+side][i],
                (m_fRef[side][i]).real,(m_fRef[side][i]).imag);
          }  
```

어떻게 이렇게 되는거지. 안 될 이유가 없잖아

포기하고 matlab보고 코드 새로 짜야하나?


### 01.30
  
일단 잘 그냥 AEC만 했을 때 잘 돌아가는 지 확인해보자

```C++
#include "Sub.h"

#define AEC_UNIT 2048

void test(){
  
  int i,j;

  WAV input(CHANNEL+REFERENCE,RECORD_SAMPLE_RATE);
  WAV output(CHANNEL,RECORD_SAMPLE_RATE);
  
  StereoANC_RLS anc(AEC_UNIT,FRAME_SIZE,CHANNEL);
  AfterProcessor ap(AEC_UNIT,FRAME_SIZE,CHANNEL);


  input.OpenFile("input.wav");
  input.Print();

  output.NewFile("otuput_test.wav");

  short wav[(CHANNEL+REFERENCE)*FRAME_SIZE];
  //double raw[(CHANNEL+REFERENCE)][AEC_UNIT];
  //double out[CHANNEL][FRAME_SIZE];
  double**raw;
  double**out;

  raw = new double*[CHANNEL+REFERENCE];
  for(i=0;i<CHANNEL+REFERENCE;i++){
    raw[i] = new double[AEC_UNIT] ;
    memset(raw[i],0,sizeof(double)*AEC_UNIT);
  }
  out = new double*[CHANNEL];
  for(i=0;i<CHANNEL;i++){
    out[i] = new double[AEC_UNIT] ;
    memset(out[i],0,sizeof(double)*AEC_UNIT);
  }

  while(!input.IsEOF()){
    input.Read(wav,  (CHANNEL+REFERENCE)*FRAME_SIZE);
   //SHIFT
   for(i=0;i<AEC_UNIT - FRAME_SIZE;i++){
        for(j=0;j<CHANNEL+2;j++){
         raw[j][i] = raw[j][i+FRAME_SIZE];
        }
     }
  //COPY
   for(i=0;i<FRAME_SIZE;i++){
        for(j=0;j<CHANNEL+2;j++){
         raw[j][AEC_UNIT -FRAME_SIZE+i] = (double)wav[i*(CHANNEL+2) + j];
        }
     }

   anc.step(raw,out);
   
   output.Append(ap.Array2WavForm(out),FRAME_SIZE*CHANNEL );

  }

}

```

AEC 만 떼다가 돌렸는데 memcpy가 안된다

```C++  
       for(int i=0;i<m_nBatches;i++){
         m_fRef[side][i].real.s = rdst[2*i];
         m_fRef[side][i].imag.s = rdst[2*i+1];
       }
```

```C++

struct SIMD_ALIGN simd_float {
    union {
        native_simd_t s;
        sfloat_t elems[SIMD_SIZE];
    };

    simd_float()
    {
		std::fill(elems, elems + SIMD_SIZE, 0);
    }

    explicit simd_float(sfloat_t init)
        :
#if defined(__AVX2__)
#ifdef USE_DOUBLE_PRECISION
        s(_mm256_set1_pd(init))
#else
        s(_mm256_set1_ps(init))
#endif
#else
        s(init)
#endif
    {
    }

#if defined(__AVX2__)
    explicit simd_float(native_simd_t init)
        : s(init)
    {
    }
#endif
};
```
AVX2를 사용하지 않을 경우 struct를 사용할 때
simd_float data가 s에 있는데 이것과 아래의 기타 등등 때문에 struct의 데이터가 생각한 모양으로 구성된것이 아니었던것 같다
AVX2를 쓰지 않을 경우엔 직접적으로 넣어줘야하는듯.

data는 잘 들어가는 것을 확인

결과 값이 전부다 nan 나는것은 어디서 나는 문제인가. native에서의 문제는 보류하고 Windows에서 간단한 테스트 시도중  

1초정도만 노이즈 캔슬링하기 전의 소리가 나다가 치직하면서 소리가 없어짐 . -> 출력값이 기하급수적으로 상승한다.? why? 
-> Threading 문제 각 **FRAME**에 대해 독립적이지 않았음.  
=> 윈도우에서는 작동함.

그냥 AVX를 같이 쓸수 있도록 할까? fma는 그냥 native하게 하도록 설정하고.
try try => 성공  

## 나의 삽질
1. RAW,VAD,AEC,INPUT_STORAGE 등의 여러 배열을 사용하면서 인자 크기 꼬임으로 인한 메모리 문제 발생  
2. test 코드의 용법을 자세히 보지 않음
3. algorithm이 각 FRAME에 WPE처럼 독립적이라 생각하고 돌렸으나, 독립적이더라도 데이터가 알맞지 않게 되었을 수도 있음.
4. FMA에 너무 쫄았다. 2줄짜리 코드로 풀리는 거였는데



### 01.31

## TODO
0. SIMD detecting cmake code
1. AEC window 크기 2048 - 512 VS. 512 - 128  -> 512 - 128
2. 결정난다면 그에 맞게 StereoAEC 코드 손보기 -> done
3. 손 본  AEC 코드로 sub_StereoAEC 제작 -> done
4. AEC + VAD + WPE 실시간 테스트

issue -> CPU가 AVX를 지원하지만 cpuinfo에 있으나 컴파일시 __SSE2__ 까지만 define 되고 AVX는 안됨. 

튜닝하기 쉽게 .config 파일에서 읽게 할까? - Json  써보자 

https://github.com/nlohmann/json  
를 쓰자

## TODO
1. demo 모드와 RT 모드의 프로세스 상의 스위칭 -> 프로그램이 많이 무거워져서 컴파일하는데 시간이 좀 걸림 -> done
2. AEC frame에 대해 쓰레딩할 수 있게 수정
3. 안쓰는 코드 + 파일 정리
4. 전반적인 모듈화 방안 구상

현재 fs : 16,000 일 때, VAD(6) + WPE(6) + AEC(6+2) cpu 640 정도 먹음 AEC 가 100정도 먹는거 같다

json - json for moderen c++ 추가함.  config.json 추가



### 02.01

## TODO
1. AEC frame에 대해 쓰레딩할 수 있게 수정 - done
2. SIMD 플래그 cmake 단에서 설정하기 ->  음..
2. 코드 정리 - done

frame에대해 AEC 가 쓰레딩 하도록 m_NPsi와 m_k 를 [tap]애서 [batch][tap]으로 하였는데
CPU 790을 찍었다. 터지지는 않는거 같은데...

속도는 [batch][tap]일경우 채널에대해서 돌리는 것이 조금더 빨랐다. 채널수랑 프로세서 수가 비슷해서(6/8) 그런것일 수도?

그래도 이렇게 하면 각 freq에 대해서 돌아가니까 서버에서 돌릴때 쓰레딩을 충분히 활용할 수 있을 것이다. 현재 24분 돌렸으나 잘 작동함  

CPU는 다 풀로 돌아가는데 이게 마이크로 단위로 찍히는 것이 아니라. 이제는 얼마나 여유가 있는지는 알기가 어려울것 같다.  

### 3
최대한 코드를 간결하게 정리해보자.  
+ 헤더에 설정값들 전처리 정의한 것들 다 없애버리자
+ run 하는 코드는 wpe랑 wpe+aec 두개만 남기자
+ SterecoAEC 의 main 코드말고는 별도 폴더에 보내버리자 - thread,waveio 관련 코드 제거  
+ sub routine 은 _1, _2로 변경, json을 쓰는 형식으로 전부 다 수정함

+ Shift & Copy 하는 inline 함수 -> 추가함 잘 작동함

### 2

SIMD 플래그 (__AVX__,__AVX2__)  는
리눅스의 경우엔 알아서 정의해준다지만 작업컴에서 AVX가 지원됨에도 __SSE2__까지만 define 되어있다.  
윈도우의 경우에는 컴파일 플래그로 AVX를 지정해줘야 __AVX__가 정의되니 의미가 없다.  
결국 사용자가 자기 CPU보고 알아서 설정하게 해야함. 

CPU를 직접적으로 접근하는 방법  

https://cboard.cprogramming.com/c-programming/95788-get-cpu-info.html  
https://en.wikipedia.org/wiki/CPUID#EAX=7,_ECX=0:_Extended_Features  
https://stackoverflow.com/questions/6491566/getting-the-machine-serial-number-and-cpu-id-using-c-c-in-linux  

일단 리눅스는 될거 같지만 윈도우에서도 동일하게 작동할지는 테스트 해봐야할듯.

asm 으로 cpu를 정보를 받아서 출력하는 코드. 이거를 cmake에서 try_run 해서 출력값을 받아서 avx,avx2 지원유무를 파악하고, 플래그설정을 하면 될거 같다. 

AVX, AVX2에 대한 값을 cmake가 받아서 플래그 설정하게 함. 

MSVC 는 asm 양식이 달라서 코드 추가함. Windows에서도 플래그를 잘 설정한다. 

Windows에서는 AEC 에서 OMP에서 쓰레딩이슈가 발생함. 같은 코드가 리눅스에선 잘돌아가지만  
MSVC는 OpenMP2.0 까지만 지원하고 기타 자기네끼리 막 바꾸는게 있으니 살살 수정하면서 길을 찾아야겠다. 



### 02..7

## TODO
1. Windows OpenMP 이슈해결.
2. RT_Input Device 번호를 받는 좀 더 깔끔한 방법.
3. 여타 최적화

### 1. Windows OpenMP 문제

이진 주석처리와 이진 critical.

```C++
#pragma omp critical
                   {
                    simd_cmplx NLpsiLNL{};
                    for (size_t tap = 0; tap < m_numTaps; ++tap) {
                        NLpsiLNL += ch.m_NPsi[batch][tap] * tapL[tap];
                    }
                    for (size_t tap1 = 0; tap1 < m_numTaps; ++tap1) {
                        simd_cmplx psiLNL{};
                        for (size_t tap2 = 0; tap2 < m_numTaps; ++tap2) {
                            psiLNL += psiL[tap1][tap2] * tapL[tap2];
                        }
                        ch.m_k[batch][tap1] = psiLNL / (simd_cmplx(ch.m_gamma[0][batch] * lambdaDBar) + NLpsiLNL);
                    }

                   }
 
```

해당 부분에서 문제가 발생, Left, right 모두 crtical 을 씌어서 조치는 하였으나 정확한 원인은 모르겠다.  
simd_cmplx가 local scope적용이 안되는지 기타 오버로딩된 연산자에서 충돌이 발생하는 것인지. gcc 와 msvc의 openmp 버전차이로 인한 것인지, 비주얼 스튜디오의 문제인지 모르겠다.    
퍼포먼스 차이는 크게 나지 않으므로 일단은 지금 상태로 우선순위를 미뤄두겠다.

+ config.json 관리 서브루틴 만듦
+ 최근 사용 device 번호를 config.json에 저장
+ 필요시 리스트를 새로 받고 번호를 설정할 수 있도록함

## TODO
2. AEC 쓰레딩 이슈관리(보류)
4. 파일 구조 정리
3. 알고리즘 다 inplace로

### 02.08


## DONE
1. 에러 - WAV::Read //  record_input Windows + jsone
3. 모듈화

### 모듈화 
input > raw(ch + rf) 
raw(c+r) > hann(in) > raw(c+r)  
raw(c+r) > fft(in) > raw(c+r)  
raw(c+r) > aec(in) > raw(c)  
raw(c) > vad(x) > pool(c)  
pool(c) > process(in) > pool(c)  
pool(c) > ifft(in) > pool(c)
pool(c) > hann(in) > pool(c)
pool(c) > output  

* note : shift & copy 는 전체 퍼포먼스에 유의미한 영향을 미치지 않는다는 전제.   

### Windows RTaudio  
같은 코드가 리눅스에선 작동하지만 윈도우에서는 안돌아간다. Rtaudio 기본 빌드된 rtaudio는 잘 작동함. 
cmake와 RT_Input을 살펴봐야겠다.  

CMake는 다를바가 없고 ASIO 파트는 다 넣어뒀다. 그럼 뭐가 문제일까?
record도 리눅스에서 잘만 사용했는데.. 내가 짠 코드중에서 무엇인가가 gcc 와 msvc의 동작이 다르다면?  


### 02.11

## TODO
1. Windows RtAudio input (값이 좀 이상하게 나온다) + 서피스북은 wpe 돌리기엔 너무 느림  
5. RT 소멸자 오류 - isStreaming 함수 
2. AEC 쓰레딩 이슈관리(보류)  
4. 파일 구조 정리  
4. json 예외처리
6. GUI

### Windows RTaudio  
같은 코드가 리눅스에선 작동하지만 윈도우에서는 안돌아간다. Rtaudio 기본 빌드된 rtaudio는 잘 작동함. 
cmake와 RT_Input을 살펴봐야겠다.  

CMake는 다를바가 없고 ASIO 파트는 다 넣어뒀다. 그럼 뭐가 문제일까?
record도 리눅스에서 잘만 사용했는데.. 내가 짠 코드중에서 무엇인가가 gcc 와 msvc의 동작이 다르다면?  

일단은 내 설치세팅에 기본 Record  코드를 넣어서 돌려보자 -> 잘돌아감 -> 내 record 코드가 잘못되었다.  
어떤 동작이 비주얼 스튜디오의 마음에 들지 않은 걸까  .

OS 문제였다. 현재 사용하는 서피스북의 환경에서는 버퍼크기가 288(변경불가)이다. 프로세스에 사용되는 버퍼 크기는 128이기 때문에 녹음 프로세스도 128에 맞춰서 짜둬서 버퍼크기 차로인해 제대로 작동하지 않았다. 전제 자체가 유동적인 버퍼크기여서 OS환경별로 버퍼크기가 N 인 미지수기 때문에 전반적인 RT_Input의 코드를 수정해야한다.  

#### TODO
1. stock
    + 가지고 있는 여유분의 프레임을 통해 실시간으로 stock 수를 관리한다 
    + Data stock의 값을 변경하는 모든 연산은 thread_safe하게 작동해야한다. - stock을 atomic으로 할 수 있을까?
2. Buffer : Write는 +=N 씩 Read는 -=shift_size 로 버퍼를 다루는 단위가 다르다 이에 대한 처리가 필요하다.  
    + Ring buffer의 끝에서 남은 공간이 모자를 경우의 jump
    + Write 와 Read의 offset 단위차로인한 jump한 부분에서의 예외처리

일단은 buffer의 처리가 상대적으로 쉬워보이니 먼저 해결하자



### 02.12

+ Windows RtAudio input (값이 좀 이상하게 나온다) + 서피스북은 wpe 돌리기엔 너무 느림  - done

일단은 buffer의 처리가 상대적으로 쉬워보이니 먼저 해결하자

buffer 에 값이 정상적을 옮겨지지 않음, 띄엄띄엄 값이 들어간다. 수정한 콜백함수를 기본 테스트 코드에 넣어서 돌리면 잘 돌아가는 것을 보면 콜백함수 문제는 아닌듯 함. 
해결 -> input_size와 shift_size가 섞여서 사용된 부분이 있었다. 쓰는 부분은 input_size를 사용하고 읽는 부분은 shift_size를 사용하도록 다 구분함 

stock 을 atomic 으로 함 

+ SRP 추가 - done

config.h config.cpp 추가 설정 관련항목들 여기로 옮김

### 02.13

## TODO
+ SRP - MVDR 추가 - MVDR이 잘 안된다 값이 거의 안바뀌네
+ RT 소멸자 오류 - isStreaming 함수 - Windows
+ SRP - MVDR 멀티 쓰레딩 추가
+ AEC 쓰레딩 이슈 해결(보류)
+ 파일 구조 & 코드 정리
+ 예외처리
+ GUI

서브 루틴을 통합함. 이제 전부다 module()로 돌아감
MVDR을 frame 에 대해 threading 시킴. 결과가 잘 안나와서 되는지 는 잘 모르겠지만 원래 이상하던 대로 싹다 지워지긴함  .

MVDR이 잘 안된다. weight 값이 낮아서 출력을 거의 0 으로 수렴시키고 있다.   
같은 input.wav 파일에 대해서도 Realtime_inverse_Rn_update에서 부터 Rn_inv 값이  다르다.   

MVDR 문제 위치 찾음. weight 생성시 wtemp1 wtemp2 값이 이상하게 들어간다. 쓰레딩 꺼도 그런것을 보면 연산과정에서 비주얼스튜디오와 gcc의 차이가 있는 게 아닐까 는 내일 알아보자




### 02.14

+ SRP - MVDR 멀티 쓰레딩 추가 - done
+ MVDR - 부동소수점 오류 - Rn_inv & (nom denom) - done
- SRP 확인 - done

## TODO
+ AEC 쓰레딩 이슈 해결(보류)
+ 파일 구조 & 코드 정리
+ 예외처리
+ 문서
+ GUI

MVDR값이 input short 에 /32768을 해야 같아짐. 하지만 출력이 없어진다.  
윈도우 코드에서 사용하는 WavManger가 출력시 *32768을 하는 것을 보아 입력할때도 비슷하게 할 것이라 생각함. 

부동소수점 오차로 인한 문제이다. Rn_inv 는 1이하의 값인데 nom과 denom은 최소 7자리 수이며 더 늘어난다. 복소수 나눗셈 연산과정에서 오차가 크게 생겨서 Rn_inv에는 0.00000이 들어가게 되는 것으로 보인다. 앞뒤로 스케일링하기로함.

SRP는 잘 모르겠다. 테스트 입력파일에 대해서 elev 는 90으로 받는데, azimuth 는 45를 뱉는다. -40 을 넣어야 잘 돌아감. 
테스트 파일이 달라서 그럼, SRP에 대한 테스트 파일해서는 잘 돌아감, RT로 하는 건 내일 테스트.


### 02.15

## TODO
+ 윈도우 테스트 (지속적으로)
+ AEC 쓰레딩 이슈 해결(보류)
+ 모듈화, 사용 방식에 관한 논의
+ 최적화(지속적으로)
+ 파일 구조 & 코드 정리(지속적으로)
+ 예외처리(추후)
+ 문서(추후)
+ GUI(????)

VAD WPE AEC SRP MVDR 다 윈도우에서 잘 돌아가는지 확인을 해보자. 잘 돌아감  
가끔 VAD 시작할때 안꺼지는 것만 어찌하면?

여태까지 32bit으로 컴파일 하고 있었음. 64bit을 해줘야하나?

MEMS dB 조절을 시리얼 포트에 정해진 포맷대로 write만 해주면 되는 것 같다. 날잡아서 시도해보자  
시리얼 포트 라이브러리   
  https://github.com/wjwwood/serial  

### 02.18

+ 시리얼 포트 라이브러리 테스트 - done
+ 시리얼 포트 라이브러리 사용 인터페이스 구축 - done

## TODO
+ 윈도우 안됨 - simd_float max와 minwindef.h 의 매크로 max 와 충돌함
+ Cmake : add_definition 부분 OS 구분하기
+ AEC 원본 코드와 결과 비교
+ 오디오 드라이버 설치에 관한 내용 (doc 나 설치 스크립트 등등)
+ 권한 문제 해결
+ top.h 의 sleep 정리
+ AEC 쓰레딩 이슈 해결(보류)
+ 예외처리(추후)
+ 문서(추후)
+ GUI(????)

https://github.com/wjwwood/serial 로 리눅스에서 Db 조절 테스트 성공. 사용방식을 다듬고 윈도우에서 테스트 해보자.

시리얼 포트 인터페이스 순서
1. 포트 연결확인
2. 전송 (gain 은 32dB 고정)
3. 전송확인
4. out

문제  
 - > Windows에서는 포트 이름을 COM5로 잘 받는데 ubuntu 에서는 /dev/ttyUSB0 이런식으로 되어있고 /dev/tty/~ 가 엄청 많이 있다.
기본적인 것들 다 필터링 해야함.  -> id 가 (n/a) 인 것들 거르면 됨 
 - > DSP 통신이 sudo 권한을 요구함. -> 실행 시 항상 요구하게 해야하나? 프로그램 단에서 권한을 얻을 수 있나?

문제  
 AEC 원본 코드와 결과값이 좀 다르다. 캔슬링은 되는거 같지만, specgram이 벌겋게 나오며 테스트 장치가 달라서 그런거 같아서 Windows에서 돌리려고 했으나 simd_float이 정의 안되는 에러가 나타남.. 

문제  
 윈도우에서 simd_float이 정의 안되어 있다고함. 해당 코드는 건든적이 없는데 다른데서 define 되는 건가? 
 -> 그렇다 minwindef.h 라는 윈도우의 스탠다드 헤더에  그냥 max가 정의 되어있고 이것이 simd_float 의 max 함수와 충돌한다. 아니 어떤 또라이가 스탠다드 헤더에 그냥 max를 정의해 놓은거여



### 02.19

+ 윈도우 안됨 - simd_float max와 minwindef.h 의 매크로 max 와 충돌함 -> NOMINMAX
 AEC 원본 코드와 결과 비교 - done

## TODO
+ Cmake : definition과 헤더 개선
+ FFT를 검증용(numerical)과 실사용(Ooura)로 스위칭 가능하도록 설계.
+ 오디오 드라이버 설치에 관한 내용 (doc 나 설치 스크립트 등등) 추가
+ 최적화(SRP,FFT)
+ 권한 문제 해결
+ top.h 의 sleep 정리
+ AEC 쓰레딩 이슈 해결(보류)
+ 예외처리(추후)
+ 문서(추후)
+ GUI(????)

max,min 모두 다 바꿔야함 -> 해결  
```
#define NOMINMAX
```
하면 된다.

CMAKE에서 헤더를 include로 받으면 비주얼 스튜디오에서 외부종속성으로 분류됨.  
include로 헤더를 넣지말고 직접 src에 넣고, 인클루두 했던 경로를  CMAKE 소프 파일 경로에 넣어줘야할듯. 

AEC는 왜 다를까. 

그럼 일단 AEC를 원본 껄로 돌려보자

AEC가 다른 것은 FFT 와 32767의 차이가 만든 결과.  -> 원래 쓰던 fft 가 어디서 온것인가?  
전반적으로 오차 회피나 그런거 고려해보면 전반적으로 scaling 해야할듯

numerical은 느리지만 정확도 부분에서 더 좋다. 
Ooura는 빠르지만 정확도 문제가 더 크다. 

FFT를 스위칭 할 수 있도록 구현하자. 

FFT의 종류는 top.h 로 하고 컴파일 시 설정하도록 하자. 막 스위칭할 그런 사항은 아닌듯.
2^15 스케일링은 항상 하도록 하고.

FFT 다 inplace로 두자,out은 별도 구매

Num_FFT 마저 고쳐야함


### 02.20

## DONE 
+ FFT를 검증용(numerical)과 실사용(Ooura)로 스위칭 가능하도록 설계. 
+ 최적화(FFT)
## TODO
+ Cmake : definition과 헤더 개선
+ 오디오 드라이버 설치에 관한 내용 (doc 나 설치 스크립트 등등) 추가q
+ 권한 문제 해결
+ top.h 의 sleep 정리
+ AEC 쓰레딩 이슈 해결(보류)
+ 예외처리(추후)
+ 문서(추후)
+ GUI(????)

Num_FFT는 했고 Ooura에 멀티 쓰레딩 넣고 노트북으로 테스트 한번 돌려보자. 

audio probe 시 0번 출력해야함

윈도우에서 AEC __AVX2__가 늦게 정의됨 -> 처리해야함

윈도우에서 AEC 쓰레딩 이슈, 처음 몇초는 AEC 잘 되는데 그 다음부터는 출력이 안나옴
어떤 채널은 다 잘 나오는데 다른 채널은 안나오고 불규칙하다.
```
#define _P_1 1  //초기화 -> critical
#define _P_2 0 //Norm -> 미세하게 차이가남 -> 괜찮은가? 보류
#define _P_3 1 //NLpsiLNL -> 매우 critical
#define _P_4 1 //Psi -> critical
#define _P_5 0 // 출력 -> OK
```
이 부분들이 윈도우 에서는 thread-safe 하지 않는거 같다. 스케쥴링 방식말고 뭐가 다른걸까.
cmplx에서 문제가 있는 거 같다. critical 하지 않은 부분들은 다 cmplx가 없다. adsf saadfbcv 왜 윈도우만 안되는건가



# 02.25
## DONE
+ AEC 쓰레딩 이슈 해결(시급) -> 기존의 threading 방법을 frame에 대해서 구현하자
+ Cmake : definition & 헤더 개선
+ 오디오 드라이버 설치에 관한 내용 (doc 나 설치 스크립트 등등) 추가
## TODO
+ 실시간 녹음 API 라이브러리 파일 생성 (3월까지)
+ serial port 예외처리
+ 권한 문제 해결
+ top.h 의 sleep 정리
+ config 사용 편의성(?)
+ AEC 쓰레딩 다듬기
+ 예외처리(추후)
+ 문서(추후)
+ QT(????)

 std::thread::hardware_concurrency(); -> C++11 : 프로세서수 리턴

m_nBatch가 작아서 채널*프레임에 전반적으로 쓰레딩을 해야겠다. 오버헤드가 엄청나다.

일단 내 노트북에서 12초 짜리 6+2 입력을 받아서 채널에 대해 쓰레딩 하면 3.4 초 걸리지만 프레임에 대해 쓰레딩하고 채널별로 돌면 17초 걸린다. 각 쓰레드가 10개의 iteration만 수행해서 한 입력단위에 대해 48번의 쓰레드를 생성해야 하기 때문.  
-> 물론 리눅스에선 빠르게 돌아간다.. 윈도우 OS의 쓰레드 관리에서의 문제인듯.

AEC의 Thread에 대해 functionality 는 충족하였으니 performance를 고려하자.

수십개의 프로세서를 사용하는 경우는 윈도우가 아닐것이니(믿음) 윈도우는 채널에 대해서 돌리고 리눅스는 batch에 대해서 돌리는 걸로 해두자. 
원래의 계획은 channel * batch 에다가 processor를 분배할 계획이었으나 굳이? 라는 의문이 들었다.

---

녹음 API는 WAV 파일도 써 줘야하는가? so나 a의 라이브러리 파일로 줘야하는가? 
어차피 코드 자체가 RtAudio의 예제에서 따와서 수정한 것이기 때문에 코드 자체를 주는 것에는 문제가 없지만 라이브러리로 줘야할 수도 있으니 병행하면서 진행하고, OS도 리눅스일 가능성이 크지만 윈도우일 수도 있다.  
일단은 전반적인 사항만 구현하자

---

CMake 정리를 하자  
OS 구분해서 안쓰는 파일은 컴파일 하지 않고  
불필요한 부분 제거하고 좀 더 직관적인 코드였으면 좋겠다.   
헤더파일이 비주얼 스튜디오에서 외부 종속성으로 들어간 것을 해결함   
시리얼 포트 관련된 헤더는 외부종속성으로 하고 싶은데 그러면 좀 수정을 해야할 거 같아서 그건 보류

---

Qt 리눅스에 make 하는 데 프로세서 풀로 돌려서 한시간 가량 걸려서 설치함. usr/loca/Qt~ 에 있다. 


### 02.26
## DONE
+ SRP 최적화 - 어느정도. : 비중은 크나 원인이 FFT라서 캐시 문제로 멀티 쓰레딩은 힘들다. 
+ 파이썬 돌리기 - 그냥 system() 사용하기로
## TODO
+ 서버에서 돌려보기
+ 윈도우에서 이상현상(test 시에만 AEC가 엄청 느림)
+ 파이썬과 티키타카 해야함 (1GB 정도의 로딩을 요구함 매번 할 순 없다)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월까지) - 환경에 대한 정보를 받지 아니함
+ serial port 예외처리
+ 권한 문제
+ top.h 의 sleep 정리
+ config
+ AEC 쓰레딩 다듬기(추후)
+ 예외처리(추후)
+ 문서(추후)
+ QT(????)

현재 각 프로세스별 비중  (12초 -> 17초)

process|ratio
---|---
aec|23%
vad|3%
srp|31%
mvdr|4%
wpe|30%
나머지|1%

SRP가 엄청 먹고있다. 멀티 쓰레딩 안해서 그런듯.
+ SRP에서 iFFT 비중이 50% 이다. 
+ FFT를 Numerical에서 Ooura로 바꾸면 40%의 성능 향상

SRP를 멀티 쓰레딩하면 FFT 때문에 캐시 히트가 작살이나서 퍼포먼스 박살남.  선형으로 Ooura로 돌려서 최대한 타협해야할듯

조정 후 (12초 -> 12초)

process|ratio
---|---
aec|30%
vad|3%
srp|24%
mvdr|1%
wpe|36%
나머지|0.3%

윈도우에서는 (12초 -> 29초)

process|ratio
---|---
aec|59%
vad|1%
srp|17%
mvdr|1%
wpe|18%
나머지|0.8%

aec가 비정상 적이다. 
aec만 돌리면 3초 걸리는데 test 로 돌리면 17초 걸린다.? why? 

윈도우에서 총 CPU 에서 15% 정도 놀고있다. Cache 의 문제라 생각하고있다. 

---

데모 프로그램에 얹을 python 코드는 numpy와 텐서플로우를 사용한다. 

python.h 가 없다. python-dev 는 설치가 돠어있다. 설치가 잘 못 된거 같아서 

```
sudo autoremove python-dev
sudo apt-get install python-dev
```

를 해준다.  

```
locate python.h
```
를 하니 위치가 나옴. 링크에 어려움을 겪고있다. 

다 때려치우고 system()이나 쓰자

-> 1GB 정도의 로딩을 요구. system()은 힘들듯. 

두가지 방안이 있다.
1. embedded python in C++.
2. python과 C++ 프로세스간 통신.

2. 는 송수신 대기가 리소를 잡아먹지만 '상대적으로' 사용자체는 용이하다.
1. 은 하나의 프로세스를 사용하기 때문에 리소스를 덜 먹겠지만 환경설정이 꽤나 복잡할듯하다.(추가적인 라이브러리를 찾아야할 수도 있다)

---

프로그램 사용 구상  

사용자의 발화(오디오) -> 전처리 -> 구글 API -> 구글 API의 답변(텍스트) -> python 음성 생성 스크립트 -> 생성된 음성(오디오)  


### 02.27
## DONE
+ top.h 의 sleep 정리
+ SRP 쓰레딩
+ 파이썬 in C++ - 리눅스만
+ 포매팅
+ 윈도우 빌드 에러 해결 
+ Windows Python in C++
## TODO
+ OMP SCHEDULING UNIT 전역화
+ 서버에서 돌려보기(서버 안됨)
+ 윈도우에서 이상현상(test 시에만 AEC만 엄청 느림)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월까지) - 환경에 대한 정보를 받지 아니함
+ serial port 예외처리
+ 권한 문제
+ config
+ AEC 쓰레딩 다듬기(추후)
+ 예외처리(추후)
+ 문서(추후)
+ QT(????)

---
SRP시 FFT를 쓰레드 수 만큼만 가지고 있고 자기 쓰레드 번호로 FFT idx에 접근하면 되지 않을까

기존 

process|ratio
---|---
aec|28%
vad|3.5%
srp|25%
mvdr|1.3%
wpe|38%
나머지|0.3%

크게 도는 친구들 쓰레드 수만큼 할당하고 돌게하고 기타 등등했지만 근소하게 시리얼하게 돌리는게 빠르다. 
하지만 이 아이디어를 다른 알고리즘에 적용시켜서 memory locality를 향상시킬 수 있을 듯. 

-> 적용이 생각보다 오래 걸릴듯하다. 어떤 변수가 중간변수라 새로 생겨나는지, 어떤 변수가 값을 쌓아가야하는 것인지, 전체적으로 다 사용되는 것인지 구분해야한다.

-> 아 개 멍청 idx에 쓰레딩한다고 idx for문을 만들었는데 i,j를 그대로 둬서 원래 연산을 15배 돌리고 있었다. 
해결함

수정 후 

process|ratio
---|---
aec|33%
vad|3.8%
srp|17%
mvdr|1.4%
wpe|40%
나머지|0.3%


---

서버의 내 폴더에 접근이 안됨 input/output error 가 뜨는 데  
찾아보니 물리적 오류라고함. 당장 어찌하기는 힘들 듯

---
RtAudio API 분리중. WAV도 넣어 줘야하나? 일단은 CMake로 할 수 있게 작성함.  
API 구조 자체가 데모 프로그램 돌릴 용도로 되어있어서, 조정이 많이 필요할듯  

---

https://docs.python.org/2/extending/embedding.html

```c++
#include <Python.h>

int
main(int argc, char *argv[])
{
  Py_SetProgramName(argv[0]);  /* optional but recommended */
  Py_Initialize();
  PyRun_SimpleString("from time import time,ctime\n"
                     "print 'Today is',ctime(time())\n");
  Py_Finalize();
  return 0;
}
```

파이썬 돌리기 성공함

```CMake
find_package( PythonLibs 2.7 REQUIRED )
include_directories(${PYTHON_INCLUDE_DIRS})
list(APPEND LINKLIBS ${PYTHON_LIBRARIES} )

```
이면 된다. 
남은거는 다른 스크립트에서 import가 되는지랑

파이선 코드의 모든 줄을 "<>\n"로 포장해서 코드에 넣어주는것..

-> 윈도우에서 CMake가 파이선 경로를 못 잡는다. 라이브러리 파일이 존재하는 것은 확인 하였다.  
https://stackoverflow.com/questions/32550539/cmake-trouble-with-python-on-windows  
프로젝트는 32비트인데 64비트 파이썬만 있어서 그런가? 

윈도우에 32bit python2.7 설치해보자

---

윈도우 빌드가 안된다. 
오디오 파일은 포맷해서 문제이며 json.hpp 는 전처리가 _snprintf 가 문제다. 
이는 python.h 가 sprintf를 재정의해서 발생 

```
#define HAVE_SPRINTF 
```
를 컴파일시 넣어주자. 또한 비주얼 스튜디오로 디버깅시  
Python27_d를 요구하는데 릴리즈 모드로 하거나 Python27.lib을 복사해서 두면 될듯


### 02.28
## DONE
+ 헤더 의존성 정리 - > json.hpp, Python.h, FFT
+ 구글 API 적용
## TODO
+ PythonLibs 경로 - 어디까지 해줘야하나 
+ Windows 10에서 구글 API에 받은 스트링의 인코딩 문제
+ 구글 API와 모듈 결합 - 모듈 구조에 변화가 있어야 하나
+ OMP SCHEDULING UNIT 전역화 - 최적의 값은?
+ 서버에서 돌려보기(현재 서버ㅏ 안됨)
+ 윈도우에서 이상현상(AEC 엄청 느림)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월까지) - 환경에 대한 정보를 받지 아니함
+ 64bit 전환(해야하나?)
+ serial port 연결 시 권한 문제(리눅스)
+ config 사용 환경 개선
+ AEC 쓰레딩 다듬기(추후)
+ 메모리 locality 향상
+ 예외처리(보이면)
+ 문서화(틈틈히)
+ QT(????)

---

Python.h 와 json.hpp 의 충돌 발생. 헤더를 너무 광범위하게 둔 것 같다. 
Sub를 bottom 밑에 두지말고 옆에 다 두자.  
알고리즘 관련된 것들과 유틸을 분리하자.  -> 유틸관련된 헤더를 Sub 에서 Include 하기로함. 

---

RtAudio API 에 실시간과 타이머를 구분하게 해야하나? 그리고 일단 테스트도 해봐야함

---

구글 클라우드 API

sogang_iip 계정 연동  
프로젝트 개설후 해당 프로젝트에 해당하는 키로 API 연결하는듯
파이썬으로 돌아가며 아나콘다 3 사용중

현재 perception 서버에서 테스트  

파이썬 2.7  

```
pip install google-api-python-client
```

이건 아닌듯

클라우드 api 를 설치해야함 

```
sudo pip install google-cloud
sudo pip install google-cloud-speech
```

인증문제가 발생

구글 클라우드 API 는 

```
GOOGLE_APPLICATION_CREDENTIALS 
```
에 있는 json 파일을(구글 클라우드 API 에서 받은 계정키 파일)
을 찾아서 인증함. KEY 일단 넣었음. private repo라서  

서버에서 동작 확인함. 굳이 서버에서 돌릴필요는 없었다.  

이제 모듈에서 VAD 가 끝나면 구글 API에 보내도록 연결해야함

---

윈도우에서 파이썬 경로 문제 발생   
프로그램은 python 2.7 - 32bit 을 사용하는데 
노트북의 환경은 3.6 64bit를 기준으로 구성되어 있다. 

pip 도 3.6 껄로 되어있고. CMake 가 3.6을 잡게 하려는데 잘 안됨.  
static으로 전환 할까. 

find_package 할 떄 어떤 값들이 적용되는듯. find_package 시점에서 3.6을 찾도록 해야겠다

이건 도큐먼트를 잘 만들어야 할듯

---

구글 API에서 받은 문자열이 윈도우 터미널은 cp949를 써서 utf-8로 인코딩하면 깨지는데 뭘로해야 안깨지는 지 찾는 중.  






### 03.04

## DONE
+ 리드미 한글화 
+ 64bit 전환(어셈블리 코드)
+ PythonLibs 경로 - 이건 그냥 유저 단에서 하자 
+ 윈도우에서 이상현상(AEC 엄청 느림) - SRP원인으로 추측
+ AUX_DCICA 일단 코드만 통합  
## TODO
+ OMP SCHEDULING UNIT 전역화 - 최적의 값은?
+ config 사용 환경 개선 ->  각 모듈에 관한 item으로 
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ AEC 쓰레딩 다듬기
+ Windows 에서 구글 API에서 받은 스트링의 인코딩 문제
+ 파이썬 스크립트 내 경로 관리
+ 구글 API와 모듈 결합 - 모듈 구조에 변화가 있어야함
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)
+ GUI - QT(???)

---

어차피 내부적으로 사용할 것이니 리드미는 다 한글로 하자

---

비주얼 스튜디오에서 64bit는 inline_asm이 안된다고 한다.  
http://www.sysnet.pe.kr/2/0/1759  

1. cpuid 루틴만 34bit으로 빌드하거나  
2. 다른 방법을 찾아보자  
https://www.codeproject.com/Articles/11037/Using-Assembly-routines-in-Visual-C  

3. 이미 구현된 asm 루틴이 존재  
https://stackoverflow.com/questions/1666093/cpuid-implementations-in-c  

3을 선택함. 

--- 

윈도우에서 파이썬 경로 만지작 거리던거 cpuid push 하면서 같이 넣어버림. 이참에 정리하자. 

---

windows 에서 stereo AEC 퍼포먼스를 처리하자. 문제가 없다?

process(12s 6ch)|time|ratio
---|---|---
aec|3.2s|28%
vad|0.4s|3.5%
srp|1.9s|17%
mvdr|0.3s|2.2%
wpe|4.6s|40%
나머지|0.9s|0.3%

32bit -> 64bit 말고는 한게 없는거 같은 데, 문제가 없어졌다. ???
메모리는 10MB정도만 쓰는데, 32bit이랑 64bit 일 때랑 명령어차이가 퍼포먼스에 영향을 주는가?  

https://stackoverflow.com/questions/324015/supplying-64-bit-specific-versions-of-your-software

### Architectural benefits of Intel x64 vs. x86

+ larger address space
+ a richer register set
+ can link against external libraries or load plugins that are 64-bit

### Architectural downside of x64 mode

+ all pointers (and thus many instructions too) take up 2x the memory, cutting the effective processor cache + + + size in half in the worst case
+ cannot link against external libraries or load plugins that are 32-bit

더 많은 레지스터를 통한 더 깊은 최적화 > 2배의 인스트럭션 크기 

인가?

32bit 으로 빌드하여 보았으나 차이가 없다. 

이전 기록들을 살펴보니 srp 쓰레딩 시점부터 발생한 문제.  
처음 SRP 쓰레딩시 잘못하여 굉장히 쓰레드를 많이 생성시킴. 여기서 캐시를 다 잡아 먹어서  
AEC에 사용될 캐시가 부족했던 것 인듯

---

OMP SCHEDULING 단위를 전역화 시키자. 각 프로세스에 알맞은 쓰레드/프로세서 수를 찾고 설정하는 방법을 고안하자. 

OMP는 전처리 구문으로 구현 -> 컴파일 타임 때 값을 정해줘야함 -> 컴파일 타임에 프로세서 수를 받아와야함. 
OS 별로 다를 수 있다. 
try_run 으로 CMake 단에서 정의를 넣어 줄 수도있다.

딱히 정의된 매크로 같은 건 없는 듯. 
```
unsigned concurentThreadsSupported = std::thread::hardware_concurrency();
```
를 써서 CMake 단에서 받는 방식을 취하자. 

cpuid 쓰던 대로 프로세서 수에 대한 값을 출력받아서 정의해주는 것으로..



---

여담, gem5 를 쓰면 cpu cache 사용에 관한 정보를 얻을 수 있지 않을까?  

---

AUX_DCICA 통합중..  
잘 안됨. 코드는 통합함.


### 03.05
## DONE
+ OMP SCHEDULING UNIT 전역화

## TODO
+ config 사용 환경 개선 ->  각 모듈에 관한 item으로 
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ AEC 쓰레딩 다듬기
+ Windows 에서 구글 API에서 받은 스트링의 인코딩 문제
+ 파이썬 스크립트 내 경로 관리
+ 구글 API와 모듈 결합 - 모듈 구조에 변화가 있어야함
+ OMP SCHEDULING UNIT  최적값 구하기
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)
+ GUI - QT(???)

---
중간에 에러나서 로그 날림   
OpenMP 최적화에 대한 조사 진행  
https://koobh.github.io/2019/03/05/about-OMP.html
---

```
response = client.recognize(config, audio)
```

recognize(config, audio, retry=<object object>, timeout=<object object>, metadata=None)[source]
Performs synchronous speech recognition: receive results after all audio has been sent and processed.

Example

```
>>> from google.cloud import speech_v1
>>> from google.cloud.speech_v1 import enums
>>>
>>> client = speech_v1.SpeechClient()
>>>
>>> encoding = enums.RecognitionConfig.AudioEncoding.FLAC
>>> sample_rate_hertz = 44100
>>> language_code = 'en-US'
>>> config = {'encoding': encoding, 'sample_rate_hertz': sample_rate_hertz, 'language_code': language_code}
>>> uri = 'gs://bucket_name/file_name.flac'
>>> audio = {'uri': uri}
>>>
>>> response = client.recognize(config, audio)
```
Parameters:	
config (Union[dict, RecognitionConfig]) –
Required Provides information to the recognizer that specifies how to process the request.

If a dict is provided, it must be of the same form as the protobuf message RecognitionConfig

audio (Union[dict, RecognitionAudio]) –
Required The audio data to be recognized.

If a dict is provided, it must be of the same form as the protobuf message RecognitionAudio

retry (Optional[google.api_core.retry.Retry]) – A retry object used to retry requests. If None is specified, requests will not be retried.
timeout (Optional[float]) – The amount of time, in seconds, to wait for the request to complete. Note that if retry is specified, the timeout applies to each individual attempt.
metadata (Optional[Sequence[Tuple[str, str]]]) – Additional metadata that is provided to the method.
Returns:	
A RecognizeResponse instance.

Raises:	
google.api_core.exceptions.GoogleAPICallError – If the request failed for any reason.
google.api_core.exceptions.RetryError – If the request failed due to a retryable error and retry attempts failed.
ValueError – If the parameters are invalid.

---

class google.cloud.speech_v1.types.RecognizeResponse
The only message returned to the client by the Recognize method. It contains the result as zero or more sequential SpeechRecognitionResult messages.

results
Output only. Sequential list of transcription results corresponding to sequential portions of audio.

results
Field google.cloud.speech.v1.RecognizeResponse.results

---

class google.cloud.speech_v1.types.SpeechRecognitionResult
A speech recognition result corresponding to a portion of the audio.

alternatives
Output only. May contain one or more recognition hypotheses (up to the maximum specified in max_alternatives). These alternatives are ordered in terms of accuracy, with the top (first) alternative being the most probable, as ranked by the recognizer.

channel_tag
For multi-channel audio, this is the channel number corresponding to the recognized result for the audio from that channel. For audio_channel_count = N, its output values can range from ‘1’ to ‘N’.

alternatives
Field google.cloud.speech.v1.SpeechRecognitionResult.alternatives

channel_tag
Field google.cloud.speech.v1.SpeechRecognitionResult.channel_tag

---


### 03.06

## DONE
+ 파이썬 스크립트 내 경로 관리 - 상대경로로 수정함 
+ tts 모델을 perception에 저장함, 수정된 스크립트 받음. 
+ tts 코드만 합침
+ DCICA 비교용 출력 받음
+ 실시간으로 GCP 사용
## TODO
+ 실시간 GCP 이슈 - 터미널 입출력, 메인 쓰레드 점유, 짧은 단어 인식 문제(어디 문제인가)
+ Windows 에서 구글 API에서 받은 스트링의 인코딩 문제
+ 구글 API와 모듈 결합 - 모듈 구조에 변화가 있어야 하나? 
+ tts 스크립트 수정(사용 환경 구축해야함)
+ DCICA 최적화
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ 쓰레딩에 대한 연구
+ OMP SCHEDULING UNIT  최적값 구하기
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)
+ GUI - QT(???)
+ config 사용 환경 개선 ->  각 모듈에 관한 item으로 (GUI여야 이게 편함)

---

DCICA 최적화 준비  
functionality test용 데이터를 받아서 기능을 유지하면서 진행하자   
일단 clang 으로 포맷팅 먼저해둠   

---

tts 하시는분 발표준비로 연구실 잘 못나와서 파일 메일로 다 받기로함. 코드 정리는 해놨다고 하니.  
받아서 테스트 해보자.

아.. 코드 받았고 서버 접속도 된다. 문제는 
프로젝트 + 새로받은 코드 + 서버에 있는 1.2기가의 학습 데이터  

를 해야함. 

분석할라면 코드 다 봐야할 거 같은데,
한번 설명을 듣는 편이 훨씬 생산적일 듯 하다.

알아야하는 것이

1. 경로관련 (데이터 로드, 출력파일 생성 등 모든 경로 관련 부분)
2. 로딩 
3. 프로세스

경로 있는 부분이
상대경로로 되어있는데, 2군데 정도만 수정하면 될듯하다. -> 파일 배치 어떻게 할 건지 생각해봐야함 

모델은

new1은 손석희  
kss 는 여성  
ma 는 spectrogram 을 wav로 변환하는 것이라 한다. 

코드를 받았는 데, 다 하나의 함수에 들어있다. 클래스의 생성자에 로딩과정을 넣고  
모든 프로세스를 구분해야겠다. 

일단 환경부터 구축하는 데 걸릴듯함. tts 환경에 대한 정확한 정보도 모르지만 큰 문제는 없다.
하지만 환경을 구축할만한 장비가...

---

원활한 테스트를 위해서 파일 경로를 세팅해줘야한다. 

 C < - >  python  
데이터를 주고 받는 법을 알아야하나? 

+ https://docs.python.org/2/extending/embedding.html  

그냥 os 모듈로 스크립트 위치 받아서 상대경로 조작해서 경로 설정하자.

테스트 해봄.

perception server에서 동작을 확인. 


---

GCP 의 출력 문자열 인코딩에 대해 생각해보자.   
해당 결과물 write 해야하는데, 그것이 호환이 안된다. -> 호환되는 타입으로 변경하거나 호환되는 write 를 찾으면 된다.  
print 자체는 잘 되지 않는가. 

일단 우분투에서는

```
outstr = (result.alternatives[0].transcript).encode("utf-8")
```
이면 되지만 이는 윈도우에서는 안됨. 일단 리눅스는 되니까. 보류.

---

구글 API 를 합쳐서 동작하는 것을 생각해보자.  
1. 당연히 VAD를 사용해야한다. 
2. 각 VAD 발화구간당 별도의 파일을 출력해야한다. 
3. 발화구간이 끝나면 생성된 출력에 대해 구글 API를 사용해야한다. 
4. 이 모든 과정은 실시간으로 돌아가야함. 

발화 구간.. 파일은 3개정도 이름 돌려가면서 덮어 씌우고 그때그때 보내면 될듯

얹어진 GCP 작동하는 것은 확인 했지만 문제가 있다.

파이썬 인터프리터를 C 에서 열면 C가 돌아가는 터미널이 인터프리터 역할도 하게됨.
C의 터미널 입출력과 꼬이는 부분이 존재. 

원래 계속 돌아가던 프로그램 강제로 끄는 방식으로 했는데, 이젠 못함. 


또한 인터프리터가 단일 쓰레드로 돌아가는 영역에서 리턴받을 때 까지 대기하기 때문에, 낭비가 심하고  
실시간도 잘 안되게됨. 

파이썬 스크립트도 정리하고 전반적으로 gcp 서브 루틴을 정리 하고 넘어가자.



### 03.07

## DONE
+ More OOP aspect for module usage - Module Class 
## TODO
+ To send data from cpp to python.
+ 실시간 GCP 이슈 - 터미널 입출력, 메인 쓰레드 점유, 짧은 단어 인식 문제(어디 문제인가), can't shutdown
+ Windows 에서 구글 API에서 받은 스트링의 인코딩 문제
+ 구글 API와 모듈 결합 - 모듈 구조에 변화가 있어야 하나? 
+ tts 스크립트 수정(사용 환경 구축해야함)
+ DCICA 최적화
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ 쓰레딩에 대한 연구
+ OMP SCHEDULING UNIT  최적값 구하기
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)
+ GUI - QT(???)
+ config 사용 환경 개선 ->  각 모듈에 관한 item으로 (GUI여야 이게 편함)


--- 

can't type Korean for now. Will fix it tomorrow

I came up with idea of capsulizing module process

create Class Moudle.

Tried to add some OOP features. not enough per se. But roughly minimized overhead part of codes.  
next priority is usage of GCP. need to change vad procedure with each wav output for each speech.  
And need to find send data from cpp to python script. 

ex) speech 1 2 3 -> wav  1 2 3 -> gcp 1 2 3 -> result in ... not into file for now. just terminal output.  
  

---

About Terminal problem..

Maybe Run another terminal for IO purpose and leave main terminal for python(or contra), can shutdown process manually (not by sudo kill)


## DONE
+ 구글 API와 모듈 결합 - 적당히 합침.
+ AUX_IVA 모듈 넣음
## TODO
+ AUX_IVA 용 출력분리
+ AUX_IVA 정리 및 최적화
+ To send data from cpp to python.
+ 실시간 GCP 이슈 - 터미널 입출력, 메인 쓰레드 점유, 짧은 단어 인식 문제(어디 문제인가), can't shutdown
+ Windows 에서 구글 API에서 받은 스트링의 인코딩 문제 (리눅스는 utf-8 변환)
+ tts 스크립트 수정(사용 환경 구축해야함)
+ Module option validation checking sequence
+ DCICA 최적화
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ 쓰레딩에 대한 연구
+ OMP SCHEDULING UNIT  최적값 컴파일시 설정
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)
+ GUI - QT(???)
+ config 사용 환경 개선 ->  각 모듈에 관한 item으로 (GUI용)


---

언어 설정에서 한국어 지우고 재설치함. 
다시 한국어 잡힘  

---

AUX_IVA 모듈 합치는 중. 세그가 터진다.  
메모리 문제를 해결함.  할당 크기 오류.  
2채널 입력에서 음성을 분리하여 2개의 출력파일을 생성해야함.  
MODULE 옵션의 유효성을 체크하는 부분을 추가해야겠다. 

적당히 돌아가는 것 같다. AUX_IVA 자체 업데이트 중이니 엄밀한 검사는 나중에 하자.

헤닝이나 fft 같이 안 쓸 부분을 제거하자. 

프로세스를  전체적으로 프레임에 대해 돌도록함.  
sequential 하게는 테스트 검증  OMP 넣으면 오차 좀 발생 수정 요밍.  

원래 루프가 
```
채널{
  프레임{

  }
}
```
이던 걸 
```
프레임 {
  채널 {

   }

}
```

음...



## 03.11

## DONE
+ iva 시 출력 웨이브 파일을 side_1.wav 와 side_2.wav 로 분리
+ AUX에 크게는 채널에 돌고 내부에 freq에 도는 #omp 문을 여러개 넣어서 쓰레딩 
+ DCICA 쓰레딩
## TODO
+ 사용 설정에 맞게 cmake 옵션 추가와 옵션별 분할 (요구사항이 너무 많아짐)  해야 하겠지 ..
  ex. 파이 사용 유무, tts 사용 유무 -컴파일 될 소스가 어디까진 지 정해야함. 
+ To send data from cpp to python.
+ 실시간 GCP 이슈 - 터미널 입출력, 메인 쓰레드 점유, 짧은 단어 인식 문제(어디 문제인가), can't shutdown
+ Windows 에서 구글 API에서 받은 스트링의 인코딩 문제 (리눅스는 utf-8 변환)
+ tts 스크립트 수정(사용 환경 관련 - 경로)
+ Module option validation checking sequence
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ 쓰레딩에 대한 연구
+ OMP SCHEDULING UNIT  최적값 컴파일시 설정
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)
+ GUI - QT(???)
+ config 사용 환경 개선 ->  각 모듈에 관한 item으로 (GUI용)

---

일단 AUX_IVA는 2채널에 대해서 도니까.
AUX_IVA 는 2개의 출력 파일로 내보내야함.  
근데 이걸 주 procedure  내에서 하려니 출력에 관련 된 버퍼들을 다 2개로 쪼개야함. 
2채널 웨이브 출력을 만든 다음에 그걸 쪼갤까?  

AfterProcessor 에 채널 x 프레임으로 저장되어있고 그것을 꺼낼 때, 웨이브 포맷에 맞게 1열로 출력함. 
이 부분을 수정하면 되지만 문제는 그러면 선언되는 웨이브 파일이 iva 를 쓸때만 1채널 짜리 2개가 되야하고 이게 mvdr일때는 채널 1짜리 출력 1. 이런식으로 케이스가 많아져서 전체적으로 코드가 난잡해 질 것같다. 차라리 깔끔하게 웨이브 클래스에  
채널별로 쪼개는 기능을 추가하는 것이 전체적으로 코드는 깔끔할듯. 대신 일이 많겠지만. 

WAV 클래스에
 
```C++
void WAV::SplitBy2(const char*,const char*);
```

추가함.  
iva 사용시 output.wav 를 sied_1.wav 와 side_2.wav 로 분리.  

---

AUX_DCICA를 테스트 해보자. 사용할 스티어링 벡터와 비교할 결과 파일-cmplx- 을 사용하자. 

출력비교 하는 test 함수 만듦. 유효성 확인함.  근데 seg  가 뜨네. 할당해제 부분에 메모리 침범이 있었다. 해결함.  
for 문이 freq에 대해서 도니까 쓰레딩 넣기는 쉬울듯 하다. 
 
freq에 대해 쓰레딩 추가하고 값 같게 나오는 것 확인함. 

### 03.12

## DONE

## TODO
+ 사용 설정에 맞게 cmake 옵션 추가와 옵션별 분할 (요구사항이 너무 많아짐)  해야 하겠지 ..
  ex. 파이 사용 유무, tts 사용 유무 -컴파일 될 소스가 어디까진 지 정해야함. 
+ To send data from cpp to python.
+ 실시간 GCP 이슈 - 터미널 입출력, 메인 쓰레드 점유, 짧은 단어 인식 문제(어디 문제인가), can't shutdown
+ Windows 에서 구글 API에서 받은 스트링의 인코딩 문제 (리눅스는 utf-8 변환)
+ tts 스크립트 수정(사용 환경 관련 - 경로)
+ Module option validation checking sequence
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ 쓰레딩에 대한 연구
+ OMP SCHEDULING UNIT  최적값 컴파일시 설정
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)
+ GUI - QT(???)
+ config 사용 환경 개선 ->  각 모듈에 관한 item으로 (GUI용)

---

뭐 할거가 있었는데 계단 올라오면서 까먹어버렸다. 간단하지만 좋은 변화를 낼 아이디어 였는데..

---

일단 GCP 관련 이슈부터 차근차근 해결하자 지리한 문제들이지만 해결을 해야한다.
일단 리눅스에서 테스트 할꺼고 윈도우 환경에서는 그때 그때 os check 하면 되니. 
일단 OS check 하는 구문 부터 넣자 

일단 

```python
# linux 는 'posix', Windows 는 'nt' 가 나온 다고함
os.name
```

로 구분함.  

콘솔을 2개 열어서 하나는 명렁부 하나는 파이썬 인터프리터용 으로 두려고 했으나, 이것은 OS 의존성이 강한 부분이므로 우분투와 윈도우 둘 다 하려면 일이 두배가 된다. 

일단 간단하게 버튼 하나만 달린 GUI를 연결해서 마이크 동작시에만 사용하게 할까?  그럼 또 프로젝트에 QT 에 대한  
Dependency가 추가 되버리는데... 가면 갈 수록 프로그램이 무거워진다.  
생각을 해봐야한데.. 생각해봐야하는 이슈들을 다 미뤄나서 생각해봐야하는 이슈만 남았다. 

파이썬에 cpp 데이터를 보내는 것은 파일명만 다루면 되니까. 문자열 조작으로 해결하자. 

생각해보니 어찌 되었든 간에 GCP 가 돌 인터프리터는 별도의 쓰레드에서 돌아야함. 

GCP procedure 시작할 떄 파이썬 쓰레드 열고 콜백으로 호출하는 방식으로 가자. 이슈가 많이 발생할 듯.  




## DONE
+ CMake GCP 옵션 추가 - 빌드 옵션
+ 파이썬을 별도의 쓰레드에서 돌게함
+ 실시간 GCP 이슈 - 메인 쓰레드 점유,
+ Windows 에서 구글 API에서 받은 스트링의 인코딩 문제
## TODO
+ 사용 설정에 맞게 cmake 옵션 추가와 옵션별 분할 (요구사항이 너무 많아짐)  해야 하겠지 ..
  ex. 파이 사용 유무, tts 사용 유무 -컴파일 될 소스가 어디까진 지 정해야함. 
+ 실시간 GCP 이슈 - 터미널 입출력, 짧은 단어 인식 문제(어디 문제인가), can't shutdown
+ tts 스크립트 수정(사용 환경 관련 - 경로)
+ Module option validation checking sequence
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ 쓰레딩에 대한 연구
+ OMP SCHEDULING UNIT  최적값 컴파일시 설정
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)
+ GUI - QT(???)
+ config 사용 환경 개선 ->  각 모듈에 관한 item으로 (GUI용)

---

cmake에 GCP 옵션 추가 중,

옵션 추가함.

---

GCP 파이썬 경로문제 해결함

GCP 잘 안됨. GDP 프로시저에서 출력파일이 0이 나온다. 
전반적으로 꼬여있었음.  
그래고 6채널 출력파일은 GCP가 안 받아준다.. 

--- 

파이썬 클래스를 별도로 생성하고 그걸 procedure에 넣자.  
해당 클래스를 돌리는 쓰레드를 생성하고  vad 구간이 끝나면 콜백함수를 호출하자.  

GCP 클래스 구축중...

일단 cpp 에서 쓰레드를 다루는 다양한 형태에 대해서 공부해보자..

+ https://thispointer.com/c11-multithreading-part-8-stdfuture-stdpromise-and-returning-values-from-thread/  
+ https://stackoverflow.com/questions/25814365/when-to-use-stdasync-vs-stdthreads   
   

+ c++11
+ packaged_task
+ future & promise - 원하는 기능이 아님. 
+ async - 스레드 수가 일정 갯수 넘어가면 메인 쓰레에서 돌게함 .. 별루..

쓰레드에서 GCP 를 하니까 response 부분에서 반응이 없다. print 찍히는 거 봐서는 돌아는 가는거 같은데. except도  
안뜨고 그냥 멎어버리네ㅐ..

+ https://stackoverflow.com/questions/3800435/why-does-google-app-engine-support-a-single-thread-of-execution-only

구글 API를 쓰려면 싱글 쓰레드여야 함..? 
Py init 한 쓰레드하고 Transcribe 하는 쓰레드가 달라서 안 받아 준듯.  
같은 쓰레드에서 돌아가도록 했다. 

이제 전반적인 사용양식을 다듬어야함 .  
발화구간이 GCP 처리하는 도중에 끝나버릴 경우엔 어떻게 하나?   

일단 GCP 하는 도중에는 블록하기로함..  

---

노트북에서 테스트 하려했는데 파이썬 경로가 또 꼬인것 같다. python37.dll 을 몾찾는다.  
파이썬 다운로드에서 압추파일로 되어있는 설치 파일에서 추출해서 사용함..

---

인코딩이 문제가 없어졌다. 아마 저번에 한글이 안쳐졌을 때 만지작거리면서 우분투 기본언어가 한글이 되었는데, 그러면서 OS 기본 인코딩이 바껴서 출력이 잘 되었던듯.  

일단 인코딩 문제는 케바케가 심한거 같으니까 notation 정도만 넣어두고 놔두자...



### 03.14
+ cmake QT5 관련 항목추가  
+ 최소한의 OFF 기능만 가지는 GUI 서브루틴 추가 - 테스트용 - 
## TODO
+ 사용 설정에 맞게 cmake 옵션 추가와 옵션별 분할 (요구사항이 너무 많아짐)  해야 하겠지 ..
  ex. 파이 사용 유무, tts 사용 유무 -컴파일 될 소스가 어디까진 지 정해야함. 
+ 실시간 GCP 이슈 - 터미널 입출력, 짧은 단어 인식 문제(어디 문제인가), can't shutdown
+ tts 스크립트 수정(사용 환경 관련 - 경로)
+ Module option validation checking sequence
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ 쓰레딩에 대한 연구
+ OMP SCHEDULING UNIT  최적값 컴파일시 설정
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)
+ GUI - QT(???)
+ config 사용 환경 개선 ->  각 모듈에 관한 item으로 (GUI용)

--- 

일단 GCP 시행시 block 을 해야함.  
열려있는 스트림을 닫는 건 비용이 크니까, GCP가 돌아가는 동안은  
버퍼를 그냥 버려버리자. 

실시간으로 작동하는데 큰 이상이 없음. gcp 걸리는 거는 케바케가 심한듯하다. 


---

IO control 을 위한 최소한의 GUI를 달아보자

cmake QT5 관련 항목추가  
최소한의 OFF 기능만 가지는 GUI 서브루틴 추가 - 테스트용 - 



### 03.15

기록 날라감.. 한번도 저장을 안했다니..  


### 03.16

GUI 계속 ..

위젯 전환은 성공함. 
콤보박스도 됨.  
가장 큰 문제는 라이브러리 파일을 프로젝트 폴더에서 받아오는 건데.. 
이게 별도의 배포 프로그램을 사용해야하게 해뒀을 것 같진 않는데. 인식하는 데 뭔가 더 필요한 걸까? 
direct하게 cmake에서 전체 경로와 파일명을 명시해서 링크해보자


### 03.19 
최근엔 여기서 작업중 https://github.com/kooBH/Qt5_practice  

라이브러리 파일을 빼서 리눅스에서 빌드 성공.
윈도우 QT 라이브러리 파일을 빼서 빌드 시켜보자.
성공하였으나 몇가지 처리해야할 사항이 존재

이제 본격적으로 UI를 구성하자  
  

### 03.20

## DONE
+ Qt5 Framework 라이브러리 추가
+ Module UI
## TODO
+ 출력파일 관리 구문 추가
+ 사용 설정에 맞게 cmake 옵션 추가와 옵션별 분할 (요구사항이 너무 많아짐)  해야 하겠지 ..
  ex. 파이 사용 유무, tts 사용 유무 -컴파일 될 소스가 어디까진 지 정해야함. 
+ 실시간 GCP 이슈 - 터미널 입출력, 짧은 단어 인식 문제(어디 문제인가), can't shutdown
+ tts 스크립트 수정(사용 환경 관련 - 경로)
+ Module option validation checking sequence
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ 쓰레딩에 대한 연구
+ OMP SCHEDULING UNIT  최적값 컴파일시 설정
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)

----

라이브러리 넣음. 이제 module 부분을 합쳐야함. 

Qt 포인터 이슈가 스마트 포인터로 해결할 수 있지 않을까 싶었는데  
https://stackoverflow.com/questions/48471191/using-qt-classes-with-unique-ptr  
계층문제가 있는듯. 그냥 고전적인 방법으로 가자..

음 메모리 해제 관련해서 좀 알아봐야겠다. QT가 자체적으로 하는 그런것들이 있어서  
자체적으로 하는 그런것들이 뭔지 알아야함.  일단 기능부터 넣자.

Application 에서 해제해서 그런가. Application에는 최소로 둬야하는가?

일단 그냥 delete 하는 걸로 가고. 
기본 프로세스를 수행하는 UI는 달음. 실시간 되는지 테스트 해봐야함.  
그러면 출력파일이 여러개 생성될 수 있도록 구문을 변경해야한다. 

---

헤더 파일도 숫자가 많이 늘었으니 정리하자.
일단 algorithm 폴더 파서 옮겼는데, 인클루드 다 바꾸는 건 미루려고

```CMake
include_directories(
  include
  include/algorithm
  include/util
```
넣어둠

---

Process 가 세그남. 왜?
고침, 선언하지 않은 rt에 사용하지 않을 때도 check 하는 구문에서 접근함.. 고쳤던것같은데 ?


### 03.21

## DONE
+ correct linking of shared libraries
+ Qt and Python macro conflict
+ 출력파일 관리 구문 추가

## TODO
+ GCP 도중에 마이크 입력 받지 않기 또는 비우기 (Rt_Input 수정)
+ GCP 에 UI 넣기
+ 각종 쓰레드 이슈 방지 & 코드가 너무 더러워지지 않게 하기
+ 사용 설정에 맞게 cmake 옵션 추가와 옵션별 분할 (요구사항이 너무 많아짐)  해야 하겠지 .
+ tts 스크립트 수정(사용 환경 관련 - 경로)
+ Module option validation checking sequence
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ 쓰레딩에 대한 연구
+ OMP SCHEDULING UNIT  최적값 컴파일시 설정
+ 서버에서 돌려보기(현재 서버가 안됨)
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)

---
출력파일 이름을 현재 시간으로 함.
이제 GCP 수행시 이걸 받아서 파이썬으로 넘겨줘야함. 

문자열 조작으로 처리함.

이제 각종 이슈를 생각해야할때.

+ GCP 도중에 마이크 입력 받지 않기 또는 비우기
+ 각종 쓰레드 이슈 방지 & 코드가 너무 더러워지지 않게 하기

rt 버퍼를 리셋시키는 걸 넣어야겠다. ㅇㅇ 그게 편할듯. 

---

Python.h 와 QT 를 같이 사용할 경우

Object.h 에서 
```
error : expected unqualified-id before ';' token
         PyType_Slot *slots; /* terminated by slot==0.*/
```

가 발생하는데 QT에서 slots이라는 매크로를 사용하기 때문이다. 해결 하려면
Python.h를 하는 include 하는 부분을

```C
#pragma push_macro("slots")
#undef slots
#include <Python.h>
#pragma pop_macro("slots")
```
이렇게 해주면 된다. 

---

아 시리얼 포트 할려면 sudo 가 필요한데
sudo를 쓰면 Qt가 안 잡힌다. why? 

```
/usr/lib/x86_65-linux-gnu/libQt5Core.so.5 : Version 'Qt_5.12' not found (required by ./iip_demo)
```

so.5는 쓰지도 않는데. ? why?

원래 파일도 ldd 보면 /usb/lib 에 링크 걸려있다.

standalone 이 아니었던 것인가? 

일단 .profile에 로 등록된 경로로 잡는거 같으니 지우고 재부팅하고 해보자

$PATH 하고 $LD_LIBARARY_PATH 둘 다 Qt가 없는데 /usr/lib에 링크를 걸어버리네  
테스트로 돌려본거는 링크를 임의의 위치이 잘 걸고 있다. 아
Qt 폴더 PATH를 없애니 gcc폴더에 넣어둔 라이브러리를 받네. 음..

라이브러리를 .so를 명시해 놨는데 .so를 받지를 않고 .so.5를 받네  
윈도우는 잘 받는데 lib폴더에서 .so.5를 두니 프로젝트 폴더를 우선적으로 받는다.  
.so 를 날리고 .so.5를 두고 sudo로 빌드해보자.  
이제 됨.

---


### 03.22
## DONE
+ GCP 도중에 Rt_input stop.
+ ui 작업중
## TODO
+ GCP 에 UI 넣기
+ 가끔 시작시 vad가 계속 켜져있는 현상
+ 각종 쓰레드 이슈 방지 & 코드가 너무 더러워지지 않게 하기 ( Modren 한 문법을 공부해보자.. )
+ 프로그램에서 마이크 버퍼 정리하기. 
+ 쓰레딩에 대한 연구 - 빠르게 AND 깔끔하게 
+ 사용 설정에 맞게 cmake 옵션 추가와 옵션별 분할 (요구사항이 너무 많아짐)  해야 하겠지
+ Module option validation checking sequence
+ 메모리 locality 향상(작업량 많음 -- 틈틈히 할까)
+ OMP SCHEDULING UNIT  최적값 컴파일시 설정
+ 서버에서 돌려보기(현재 서버가 안됨)
+ tts : 스크립트 수정(사용 환경 관련 - 경로), 1기가 데이터 셋 관리방법, 사용 방식 등
+ 실시간 녹음 API 라이브러리 파일 생성 (3월중) - 환경에 대한 정보를 받지 아니함
+ serial port 연결 시 권한 문제(리눅스)
+ 예외처리(생각나는 대로)
+ 문서화(틈틈히)

---

일단 GCP 돌리는 동안은 RT 가 동작하지 않아야함. 

```C
void RtAudio::startStream(); //	[inline] 	
//Public method which starts a stream.
//An RtError will be thrown if a driver error occurs or if called when no stream is open.

void RtAudio::stopStream(); //	[inline] 	
//Stop a stream, allowing any samples remaining in the queue to be played out and/or read in.
//An RtError will be thrown if a driver error occurs or if called when no stream is open.
```

+ GCP 가 도는 동안 rt는 stop
+ GCP 가 끝나는 순간을 캐치해서 start

Rt input stop / start 가능.

---

vad 가 가끔 시작하자마자 켜지고 안꺼지는 거 고쳐야할듯.  
노이즈 측정을 좀더 오래 할까?

---

매먼 sudo kill 하기 번거로우니 UI 부터 얹어 버리자. 

실시간일 때 UI 쓰면 seg.. - 고침
GCP 는 아직 seg남. wav 읽는거는 둘다 잘됨

일단 깔끔하게 쓰레드가 종료되는 부분먼저 달고 GCP를 고쳐보자. 

동작 방식에 대해 좀 생각해봐야함. Module 을 생성을 언제 어떻게 할 것인가.
Procedure는 언제 어떻게 수행할 것인가.
각각의 Widget 이 별도의 Module을 지니는가 하나의 Module을 지니는가. 
매번 생성과 해제를 할것인가. 한다면 어떻게 할 것인가... 다 생각해봐야함. 

App 에다가 Module을 달고 각각의 Widgets 들은 포인터만 받자.  
GCPW 의 label에 setText 하는 함수를 GCP 가 끝나면 호출할 수있게 건네 주자 - 건네주는 과정은 더러울듯.  

일단 Module을 KApplication에 다는 것 부터. 달음. 
정지 후 재시작 할때 터짐. -> Finish를 해서 Rt를 꺼버림. 전체적으로 프로시저를 재구성 해보자. 

일단 돌아가게는 달아 놨는데, 덕지덕지 붙여 놓음.  혐오스러운 상태다. 현재 상태에서 정리해 나가고   
굉장히 다양한 예외처리나 입출력 제한을 걸어주자. 

GCP 재시작 할때 파이썬에서 에러남. -- 인터프리터 관리 문제인듯. 

그냥 모듈이면 모듈, GCP면 GCP 따로 가야할까? 같이 가려니 많이 꼬이는데.. 정리를 지속적으로 하자.

윈도우에서 GCP - AEC / VAD 써서 테스트함. 잘 됨. 


### 03.25

---

UI Module 수행시 쓰레드가 14개
UI GCP 수행시 쓰레드가 15개 성성됨.

그냥 module 수행시 8 + 3

그냥  UI 켜두면 6

---

GCP seg를 잡자

seg 날 구석이 너무 많지만 가장 유력한 것은 Qt.  

Rt_input :: CleanUP() 에서 seg. 

CleanUP의 closeStream이 2번 호출 된다.

이중 CleanUp 제거함. 

----

모듈에서 전처리를 다 한다음 VAD 수행하게 변경.

---


프로그램의 State가 변하는 도중에 조작 못 하게 비활성화 하는 걸 추가해야  
인터프리터 이슈를 방지할 수 있을 듯.

이게 Transcribe 돌아가는 도중에 GCP 를 끄고 다시 시작하면 발생하는 듯. GCP 가 안정적인 상태가 되기 전까지는 대기하게 둬볼까.

비정상적인 입력으로 인한 에러는 넘겨도 되겠지만, 정상적으로 돌리는 것도 에러나는 걸 고쳐야함. 

gdb 로 하면 에러가나고 MPD_MINALLOC 메시지도 그냥 워닝임.  하지만

Thread 27 ... Segmentation fault 는 문제있는게 아닌가? 

https://docs.python.org/2/c-api/init.html#c.Py_Initialize  

study study..  

https://www.codevate.com/blog/7-concurrency-with-embedded-python-in-a-multi-threaded-c-application  

https://docs.python.org/2.7/c-api/init.html#thread-state-and-the-global-interpreter-lock  

비슷한 행동을 테스트 프로젝트로 해보았으나 문제가 없음. python.h 의 문제는 아닐 수도 있겠다.

### 03.26

저장 안하고 끔 ..



### 03.27

아마 GCP 와 Module의 생애주기를 같게 하고 옵션 처리를 안 해둔 걸로 기억함.  이 부분을 어떻게 할 건지 논의를 혼자서  
열심히 해야..

수행 되는 기능 

1. 전처리 알고리즘 테스트 -> wav input 을 가져야함
2. 실시간 전처리 후 GCP + TTS -> 지금 구현 중인 부분
3. 설정 조작 : json based configuration 
4. 기타 도구 : audioprobe,serial port control, just recording 

여기 서 문제는 1과 2를 합쳐야 하나? 말아야 하나? 현재는 분리된 상태로 유사하지만 미묘하게 다른 부분에서 겹치지 않기 때문에 하나의 머리에 2개의 몸통을 지닌 형상이다.  

일단은 module 하고 gcp 하고 생성과 소멸을 같이 하게 하였다. 지금은 1,2 1을 하려해도 아무것도 안하는 gcp가 돌아가는 중. 

출력도 종류가 3가지임.  
1. 하나의 통짜 output
2. vad를 적용한 통짜 output
3. 각 vad 구간 당 생성되는 여러개의 output.

여기서 GCP는 3을 사용.  
2를 없애자. 그냥 나중에 필요하면 여러개의 wav를 하나로 합치는 도구라도 추가하면 될 일.  이렇게 하면 GCP를 옵션으로 넣을 수 있을듯.  

### -> 통합

+ GCP는 cmake 옵션 부분에서 컴파일 유무가 설정되기 때문에 실제 코드상에서 gcp의 모양은

```C++

#if _GCP
if(setting.gcp){
  //somethnig with gcp
}
#endif

```

그러면 Module 부분에 GCP를 넣어 보자.
그리고 Run 부분을 통합하고  출력 형식을 정리하고 코드도 그에 맞게 정돈하자. . 

TimerProcedure 부분은 ...

__DEBUG 하는 부분을 정리하자. 타이머 부분도 __DEBUG에 넣어버리자. 그러면 이것도 통합을 할 수 있을 듯.  
이것까지 분기를 날려버리는 것은 마음에 들지 않는다.. 

CMakeLists.txt  에  dev 를 넣음.  
#if __DEUBG를 다 #ifndef NDEBUG 로 refactoring 함.  

Module 클래스 구조조정.  하나의 프로시저로 통합. 위젯도 하나 빼야겠네. 전체적으로 작업량이 많다. 
일단 날리고 Procedure()만 남겼다. 합치면서 분명히 버그나는 부분이 생겼을 테니 다 테스트 해보자.
그전에 Qt5에서는 wav 입력 받는거 호환 고려 안했잖아. Qt와 신호를 주고 받을 수 있는 인터페이스를 구축해야겠다. 

https://blog.pramp.com/inter-thread-communication-in-c-futures-promises-vs-sockets-aeebfffd2107  
한번 봐야겠다. 백그라운드 서비스 스레드라..

일단 돌아가게 부터만 정리하자.

rt->Stop() 에서 seg. 파일 할당 해제 부분 vad에 따라 적절하게 수정.  


항목 | 여부
---|---
wav input| O
rt|  O
rt + vad|  O
rt + vad + gcp| O

동작은 되는 상태로 통합함. 이제 타이머 부분을 디버그 옵션으로 설정해주자. 거의 함. '거의'


### 03.28

디버그 옵션시에만 로그 출력하게함. 안쓰는 소스 파일들 제거. 워닝 좀 지움.   
 
이제 파이썬에서 문자열을 받아서 위젯에 띄우게 해보자.

좀 더 확장된 방식을 시동하고 있는데, 테스트 프로젝트에는 성공하였으나  
합쳐서 하니 잘 안된다. import를 못함

import 됨. 테스트 도중 세그남. 그거 확인해야함

### 03.29

인자 전달에 어려움이 있었다 .

인자 object를 만들어서 callObject가 잘 안되서 그냥 callFunction 하고 포맷맞춰서 넣었더니 됨

```c++
      //pValue = PyObject_CallObject(pFunc, pArgs);
      pValue = PyObject_CallFunction(pFunc,"s",&command);
```

몇가지 예외처리 해야할 부분 - 종료 시의 경우에 따른 할당 해제, 동작시의 가능한 오류들 -이 있지만 위젯과 모듈간의 콜백을 먼저 구현하자.

1. 콜백구현
   GCP -> 모듈 -> 위젯 이렇게 신호가 가야함.
   멤바 함수의 콜백전달이 OOP의 규약 때문에 좀 생각할 요소가 있음. 인스턴시의 포인터를 넘겨서 수행하는게 좋을듯하다. 코드 종속 구조가 좀 더 복잡해지겠지만 사용 자체는 이것이 편리함. 

  종속관계가 꼬인것 같다. 서로가 서로를 인식하지 못한다.종속문제로 인해 Module과 GCP의 헤더를 합치고  
Forward Declaration을 걸고 정의부는 분리함. GCP 정의부 파일을 Module_GCP.cpp로 바꿈. GCP.h를 제거 


2. 문자열 주고 받기

콜백호출 잘 되고 있다. 이제 문자열을 보내도록하자. 문자열 잘 받음

3. 텍스트 필드에 문자열 표시
이제 잘 보이게 해보자. 



---

메모리 이슈가 있는 코드를 받음. 콜백 만든뒤 한번 돌려보자.

 



### 03.30

delete 랑 delete[] .  

---

메모리 이슈가 있는 코드는 범위를 벗어난 write 이슈

---

Setting UI 설계. 어떤걸 만들까.  

combobox로 하고.
default로는 module과 setting 그리고
config.json 를 읽어서 존재하는 모든 module에 대한 튜닝이 가능하도록 설정하자. 그러면 json에 관련된 헤더를 include 해줘야하는데, 또 종속성 문제가 발생하진 않을까 걱정.  
일단 json을 동적으로 설정해야하니, 해당하는 함수들은 테스트 프로젝트에서 해보고 합치도록하자.  

json 구조 자체를 다시 생각해봐야할듯. 

+ https://stackoverflow.com/questions/11944060/how-to-detect-target-architecture-using-cmake
+ https://docs.python.org/2/extending/embedding.html

1. Test routine
2. [CircleCI](https://github.com/kooBH/CircleCI-CMake  )
2. [FFTW](http://www.fftw.org/)
3. [ICC](https://software.intel.com/en-us/c-compilers)
4. Spectrogram
5. audio play

## LOG

### 04.01  

JSON을 모듈과 세팅 따로 두는 것 유지, 위젯도 별도로 사용할것임. 
모듈 UI 부터 GridLayout으로 달아보자. 스위치도 코드 구해서 돌려보자. 

외부에서 가져온 코드는 Qt tool을 사용하기 때문에 그냥 Qt코드만 돌리는 내 코드와는 호환이 잘 안된다.  
vtable 에러나 Q_OBEJCT 관련 에러들이 뜸. 

https://github.com/kooBH/Qt5_practice/issues/4



### 04.02  

swtich 코드 넣음. 
이제 swtich를 배치하고 각 swtich 값이 바뀔 때마다 json파일을 수정하게 하면 될듯. 
그리고 모듈도 바뀐 값을 갱신하게 해야함. 
매개변수들은 다 string ? const char*? 

Setting class 들에 load 함수들을 추가해야겠다.  


### TODO
1. gridlayout <- switch
2. json function
3. switch + json

일단 3 을 하는 중.  

switch가 잘 안되네 다 초깃값을 지정하는 게 없다. 의도적으로 클릭 이벤트를 발생시켜야하나? 초깃값을 반영하는 것이 코드상에서는 잘 안된다. 
스위치 달음



### 04.03

json 모듈 스위치 추가함. 문제는 모듈은 생성시에 json을 읽기 때문에 모듈을 재시작 하는 기능을 추가해야겠다. 
run 하는 동안은 정지 버튼 말고는 다 disable하면 관리가 용이 할듯.   
ModudleWidget에 모듈 재 시작 버튼을 달자. 

일단 KApplication이 너무 복잡하니 cpp 를 독립시키고 Run 위젯의 버튼 입력에따라 KApplication의 다른 위젯으로 가는 버튼들을 
enable(false)하면 될듯

Forward Declaration으로 종속성 문제가 해결이 잘 안됨. 음. 

---
모듈 UI만 후딱하고 쓰레드 건드리려 했는데 array.h의 가 \<array\>랑 충돌남. 이름이 불안하긴 했다. 
싹다 refactoring 하면 안됨. 선별적으로해야함. 했다가 다시 복구함

array.h를 쓰는 부분은 AEC와 memory.cpp

complex도 충돌난다 

array.h의 클래스명을 simd_array로
cmplex.h의 스트럭트 명을 simd_complex로 변경


---

활성화/비활성화 달음. 이제 모듈 재부팅 버튼을 추가해보자.

아 생각을 달리해야함. 모듈을 할당 후 재 할당 하려했으나 GCP가 모듈 안에 있기 때문에 GCP도 새로 열게 되고  
파이썬 인터프리터가 터짐..
GCP를 모듈 안에 넣지 말고 밖에서 써야겠다. 아 구조가 또 바뀌네


### 04.04 

AVX 테스트은 나중에 하고. condition variable 부터 달자.  한 헤더에 2클래스 선언하고 2개의 cpp에다 정의 하는 구조에서 global variable 쓰는게 힘드네. 종속구조.. 

함수로 주고 받자 그냥.. 
Module 이랑 GCP 내부구조 수정중. 

와  버스에러, system error 마구마구 터진다. QT 도 지 쓰레드 돌리는데 어디서 걸리는 거지 

일단 cv를 주고받는 건 되었다. 이제 외부에서 정지 시키는 것이 남음.

전체적으로 작동은 하는데, 어디서 뭐가 터질지 모르겠다. 테스트 테스트. 

일반적으로 예상되는 조작하에서는 잘 돌아감. 

### 04.05

https://www.dsprelated.com/freebooks/mdft/Convolution_Theorem.html  

N > 100 일 경우에는 FFT한 값이 실제 convolution값에 가까워 진다. 

https://en.wikipedia.org/wiki/Convolution_theorem 

지금까지는 FFT의 비중이 전체 프로세스의 .3% 정도라 비중을 두지 않았는데. 합성곱을 할 필요가 생긴다면,
이전 선대 프로젝트에서 쓰던 것들을 가져올 필요가 있을 것 같다. 

https://github.com/gogyzzz/iip_sph_pp/wiki/FFT-Performance-Comparision

이전 프로젝트 FFT 속도 비교  

https://www.dspguide.com/ch18/2.htm
```
FFT convolution is also called high-speed convolution.
```

---

stereoAEC의 이상현상을 좀 살펴보자
지금은 또 잘되네

---

디버깅하고 개발하기에는 UI를 사용하는 것이 너무 불편함.
현재 Module 클래스를 UI종속적으로 짜놨음.
필수적인 기능만 있는 Module Class를 UI_Module Class가 상속 받는 구조로 가야할듯.
GCP는 어차피 UI 없으면 조작이 힘들어서 UI에 의존하게 놔두자.

1. Essential Module Class
2. UI_Moudle Class
3. adjusting

일단 쓰레딩 하는 부분, UI하는 부분, GCP 하는 부분은 다 UI_Module로 가야함.
이러면 전체적인 멤버 함수들의 구조 또한 새로 생각해야할거 같은데,

GCP가 Module 이랑 같이 있을 필요가 없다. UI_Moudle 에다가 옮기긴 했는데 , include 구문들만 적당히 수정하면 될거 같다.

Run 함수 안에 얽혀있는 GCP 구문을 어떻게 분리하고 적용할 것인가. virtual로 하나 선언해서 오버라이딩할까. 
아니다 굳이 virtual일 필요는 없다. 

분리함. 
빌드는 되지만 lock 부분에서 살짝 꼬인듯

돌아감. GCP 를 UI사용시 필수적인 요소로 둘까? 
코드 더 더러워짐. 지금은 별로 청소하고 싶지 않다 .

---

OpenMP 적용시 
```C++
#pragma omp parallel for schedule(dynamic, OMP_FRAME) 
#pragma omp parallel for schedule(static, OMP_FRAME)
```

static,OMP_FRAME이 WPE에서 10% 가량 (4.4 -> 4.0 s) 빠르다.

---

HPX ? 

https://stellar-group.github.io/hpx/docs/sphinx/latest/html/manual/building_hpx.html
https://www.youtube.com/watch?v=js-e8xAMd1s
https://github.com/STEllAR-GROUP/hpx


 
### 04.06

전체적인 정리만 좀 하자

---

JSON에 option 항목을 추가해서 사용한 임시파일을 제거하는 기능을 추가하자. 하는 김에 관련 된 이름들 다 정리해 볼까.  
JSON 관련 방식을 바꿔야하는가. 

Config ? settings? 
https://www.quora.com/What-is-the-difference-between-the-words-setting-and-configuration-in-the-context-of-software-industry

refactoring

```
sed -i 's/VAD_Setting/ConfigVAD/g' $arg
sed -i 's/ModuleSetting/ConfigModule/g' $arg
sed -i 's/Setting/ConfigParam/g' $arg
```
Option : Remove_temp 추가

---

jarvis 서버에서 함 돌려보자. 컴파일 안됨. 

AVX 512 걸어주지 않아서 그럼.  

돌렸으나 작업컴에서 9초 걸리던게 22초 걸림. 오버헤드가 엄청난가.?  
지금 서버 사용중이라 나중에 날잡고 만지작해보자



### 04.08

일단 서버에서는 native 플래그로 놔두자.  최적화 문제는 서버 자원을 독점할 수 있는 상태가 아니므로 추후에.. 
ICC를 한번 해봐야겠는데. 음. 테스트하는 시점에는 무료지만, 실제로 사용하려면 비용도 있고 하니 것도 좀 추후에 해야할 수 도 있다.  

---

GCP하고 UI를 묶어야하나 독립적으로 둬야하나? 킵.
옵션별로 빌드 안되는 문제 해결함. 

---

서버에서 느린 것 해결함. 쓰레드 수가 하드코딩 되어있었다.  


### 04.09

GCP 정리.
상태 플래그들 atomic 으로 변환. 
label 조작 부분을 critical section 으로 - 간간히 쓰레드 관련해서 터졌었음.  

---

UI 에 색깔넣음. 언제든지 설정 가능 하게.

---

전체적으로 코드들 한번 둘러보자 임시로 두고 넘긴거 없는지.

---

HPX 적용 해보자.  
https://github.com/kooBH/IIP_Demo/wiki/HPX-log  
 

### 04.11 

HPX는 너무 오래 걸릴 거 같고 그렇게 안정적인 거 같지도 않다. 보류.  
일단 기능 구현에 당분간 치중하자. 



### 04.12

Setting UI.  
config.json 의 ConfigParam을 정수형 데이터만 있는 set으로 하고  
현재 값과 가능한 값들을 명시해서 동적으로 위젯을 구성할 수 있다. 다만
device 부분과 MODE 가 문제인데. 이거는 ConfigInput을 따로 구성해서 할 수 도 있을듯. 
이렇게 하면 포괄 적인 설정 위젯에다가 3개의 하부 설정 위젯을 넣는 형식으로 가면 될듯하다. 
일단

1. json 양식변경
2. Param 위젯
3. Input 위젯
4. 포괄적인 위젯 생성. 

---

JSON 양식 변경

```json
    "INPUT":{
        "DEVICE": 5,
        "REAL_TIME": true,
        "INPUT_FILE":"input.txt"
    },
    "OPTION":{
        "Remove_temp": true
    },
    "SETTING": {
        "CHANNEL":{
           "VALUE":6,
           "OPTIONS": [ 1,2,3,4,5,6]
        },
        "FRAME_SIZE":{
          "VALUE":512,
          "OPTIONS":[ 512,1024]
        },
        "SHIFT_SIZE":{
          "VALUE":128,
          "OPTIONS":[ 128,256 ]
        },
        "REFERENCE":{
         "VALUE": 2,
         "OPTIONS":[ 1,2 ]
        },
        "SAMPLE_RATE":{
          "VALUE":16000,
          "OPTIONS":[ 16000]
        }
    },

```

이런 형식으로 변경. 

--- 

Parameter 위젯 구성함. 
Input 위젯은 좀 생각을 해봐야겠다. 각 item이 형식도 상이하고 device의 설정을 어떻게 할 것인지 부터.. 

+ input  : string
+ device : int
+ real_time  : bool

input은 일단 놔두고 - 사용 방식을 설정해야함. -   
device는 audioprobe를 하고 그 결과를 바탕으로 설정하는데, audio hw들의 index를 값으로 사용한다. 하지만  
실제 오디오를 고를때는 이름을 보고 고른다. 차라리 세로로 2분할 하고 오른쪽에 audio probe 결과를 띄워줄까?  

device 는 case가 
1. 정상 case : 탐색된 device번호 범위 내에 설정된 device 번호가 존재.  
2. 새로운 환경 : 설정된 device 번호가 탐색된 device수 보다 클때.  

예외처리를 해줄까? 아니면 다 수동으로 하게 할까? 
체크 구문을 추가하면 될듯. 일단 audio probe 먼저 돌리고. 콤보박스의 표기되는 값과 실제로 각 선택지가 가지는 값을  
다르게 해서 -<string , int> 두면 될듯. 

### 04.13

Device는 장치 인덱스와 이름으로 콤보박스를 구성하고 값은 인덱스를 사용한다. 우측에 라벨필드를 만들어서 디테일한 정보를 띄우자. 

1. audio probe 수행
2. 우측 label에 표시
3. 받아온 장치 정보로 combobox구성

일단 audioprobe를 우측 라벨에 띄워주는 KInput위젯 틀 생성.  
콤보박스에 값을 넣음.  

이제 
1. audio probe 다시 수행하는 버튼 - 장치 연결이 불량했을 경우 - 
2. 실시간과 wav입력 모드를 전환하는 스위치
3. input파일의 이름을 입력하는 텍스트필드 -input.txt-가 default

+ note : 3을 구현하게 된다면, Module에 있는 하드 코딩된 input.txt를 ConfigInput에서 받아오게 해야한다. 

map 구조로 콤보박스 변경시 변경된 인덱스를 받을 수 있음. 그전에 초기값부터 받는걸 구현해야하는데. 초기값이 유효하지  
않을 경우의 예외처리를 고려해야한다.  

iterate 를 수행하고 없으면 디폴트를 고르는 걸로 하자. 
그리고 다른 위젯들 넣을 거 고려해서 레이아웃도 확실히 정리해야함.  




### 04.15

Qt 정적으로 멤버클래스 선언시 순서 중요. 아래에서 부터 할당해제 하기 때문에 큰 범위의 요소들을 위에서 부터 배치야한다.  
큰요소가 할당해제 되면 거기에 들어있던 요소들도 할당해제를 해주는데 작은요소를 포함된 요소를 한번 더 할당하게 되어  
이중 해제 에러가 발생한다.  

라벨 좀 넣고 레이아웃 설정함. combobox에 json 연결하자. 
Write 는 했는데, 초기값 설정하는 것도 해야겠다.  
RT Audio에서 유효하지 않은 device값으로 모듈 생성시 해제 부분에서 에러가 발생.  

---

유효하지 않은 - input이 없거나 존재하지 않는 - device number를 사용해서  Module를 생성하였을 때, 해제시 에러 발생 

```
"iip_demo" received signal SIGSEGV, Segmentation fault.
RtAudio::~RtAudio (this=<optimized out>, __in_chrg=<optimized out>)
    at /home/ffe/git/IIP_Demo/src/audio/RtAudio.cpp:216
216         delete rtapi_;
```

config.json의 이전에 사용했던 장치 번호로 모듈을 프로그램 시작 시 생성.  
이 번호가 유효하지 않을 경우 모듈을 유효한 번호의 장치로 재생성 할 때, 해제 부분에서 세그멘테이션.  
모듈 사용 방식의 바꿔야함.  

-> 재생성 버튼을 처음에는 생성 버튼으로 설정하자. 알맞은 변수를 다 입력했다는 전제하에 모듈을 생성하도록. 

모듈이 프로그램 시작 시 바로 생성되지 않고 생성 버튼 입력을 받아서 생상되게함. 장치 번호 잘 못 입력한 것은 유저의 잘못

---  

이제 실시간 스위치랑 입력 파일 명받는 텍스트 필드를 만들자.
- 텍스트 필드의 값을 어떻게 json 파일에 적용할 것인가. 

입력 파일을 관리하는 LineEdit 추가함.

스위치도 생성함. 작동 확인 

--

일단 AEC랑 WPE에 대한 test 모듈을 생성하고 젠킨스를 해보자. 
일단 설치는 default로 함. dependency 문제가 좀 많은듯 
CircleCI도 건드려보려 하는데 전반적으로 도커가 베이스로 이것저걸 초기단계 작업이 많은듯. 



### 04.16

도커를 써보자. 젠킨스든 CircleCI 든 도커가 있어야함.

http://pyrasis.com/book/DockerForTheReallyImpatient/Chapter03  
https://github.com/kooBH/CircleCI-CMake  
CircleCI는 윈도우는 지원안함. 사실 젠킨스 써도 윈도우에서 돌릴 게 아니라 안되는건 마찬가지.  

---

테스트 모듈. 모듈이라고 보단 서브루틴.   
각 알고리즘별 셈플 데이터를 비교하는 간단한 루틴들의 집합.  

---
 
### 04.17 

테스트 모듈 제작 중. 

AEC 테스트 해보는데 오차가 크게남. 웨이브가 미세하게 다르다. 원래 가지고 있던 샘플 웨이브의 환경을 현재 모름.. 
이제 기록해두자. 

일단 제대로 된 레퍼런스 파일 부터 정하자. 

AEC, WPE 테스트 모듈 구성함. 대응하는 circleCI를 어떻게 구성할까.  
libasound 패키지 때문에 도커이미지를 커스텀하게 만들어야함.. 그냥 전처리에서 제외해버릴까. 
1. 기존에 쓰던 이미지를 pull
2. 컨테이너 생성
3. 컨테이너 libasound 패키지 설치
4. 컨테이너를 이미지로 생성
5. 도커허브에 퍼블릭으로 이미지를 업로드
6. CircleCI에서 사용

or 

1. CMake에 CircleCI  옵션 추가
2. 전처리로 libasound 의존성 전처리 예외처리. 

CircleCI를 일단 해두는게 좋겠지... 

일단 기존 도커 이미지 받아둠 

---

fread 의 예외처리 - count 만큼 읽어야하는데 그 전에 EOF 일경우 - 가 하나도 없다. 넣어야지.  
http://www.cplusplus.com/reference/cstdio/fread/   

읽은게 모자라면 어떻게 하지 버리나? 그냥 오버랩하나? 버려도 상관없나? 일단 정상적인 상황에서는 shift 단위로 나오겠지..  
아닌 경우를 생각해줘야하나?  
모자라는 부분은 다 0 넣어주기로함. 



### 04.18  

하려는게 받은 이미지를 컨테이너로 run 시키고 거기에다가 apt-get install로 libasound2-dev 패키지를 설치하고 그 설치된 상태를 
이미지로 만들어야하는데. 

컨테이너 푸시 

https://nicewoong.github.io/development/2018/03/06/docker-commit-container/

Circle CI 돌리는 데 좀 걸리네. 
사실 8프로세서 풀로 돌려도 20초 걸리니까 적당한거 1개만 돌리도록 되있으면 한 160초 걸릴 꺼 같긴한데 8분 넘겼네.
너무 오래 걸리니까 그냥 fail 로 넘어감. 
써봤다 정도로 만족해야겠다. 
 
---

FFTW를 넣어보자. 는  GPL에 500만원이네..
아니다 데모용이니까 상관없지. 공개를 안하는데. 
명시만 해두자.. 

http://www.fftw.org/fftw3_doc/  
http://www.fftw.org/faq/  

이전에 해둔게 있음.
https://github.com/gogyzzz/iip_sph_pp/wiki/About-FFTW  

대략 
``` C
#include <stdlib.h>
#include <fftw3.h>
#include <math.h>
#include <complex.h>
#define N 8
int main(){
    fftw_complex *in, *out;
    fftw_plan p;
    in = (fftw_complex*) fftw_malloc(sizeof(fftw_complex) * N);
    out = (fftw_complex*) fftw_malloc(sizeof(fftw_complex) * N);
    p = fftw_plan_dft_1d(N, in, out, FFTW_FORWARD, FFTW_ESTIMATE);
    fftw_execute(p); /* repeat as needed */
    fftw_destroy_plan(p);
    fftw_free(in); fftw_free(out);
   return 0;
}
```
이런 형태로 사용함. 

일단은 속도보다는 오차가 얼마나 발생하는가를 볼 생각이니 빌드 옵션에 따른 차이는 보류해두자. 

---

3개 이상의 fft 옵션이 존재. 
CMAKE 단에서 전처리 분기를 할 수 있게 해보자.  
https://stackoverflow.com/questions/47611209/cmake-multiple-option-setting  
https://blog.kitware.com/constraining-values-with-comboboxes-in-cmake-cmake-gui/ 


http://www.fftw.org/fftw3_doc/One_002dDimensional-DFTs-of-Real-Data.html#One_002dDimensional-DFTs-of-Real-Data  


FFTW 는 plan을 생성해여 작업을 수행할 수 있다. 문제는 작업 시행시 대상 input,output 배열을 인자로 받아야한다. 
어차피 괜찮은 FFT - MKL 같은 - 애들도 다 plan을 요구하고 기존에 쓰던 FFT들도 쓰던 input과 output들에 대해 반복적인 작업을 수행하니. 음

### 사실
+ FFTW는 plan 생성시에 입력과 출력 배열을 받아야한다.
+ 기존의FFT는 입력을 내부의 temp 배열에 복사한뒤 수행하고 그걸 입력에 대입하는 것으로 inplace를 수행

### 방법 2
+ 기존의 FFT_Base의 생성자를 변경
+ FFTW에서 입력,출력 복사 2번 시행 - 내부 배열에 대해 plan 구성 -  

근데 전체 작업에서 fft의 비중이 너무 낮아서 굳이.. 라는 생각이 계속든다. 테스트만 하는거니까 달아서 돌려보고 오차나는지만 확인할까..

### 04.21

빌드하는 것만 circleCI에 달아두자.

QtGui가 OpenGL을 요구하는 거 같다. 

```
In file included from /root/project/include/Qt5/QtGui/QtGui:45,
                 from /root/project/include/Qt5/QtWidgets/QtWidgetsDepends:4,
                 from /root/project/include/Qt5/QtWidgets/QtWidgets:3,
                 from /root/project/include/K/IMAN4F_Switch.h:20,
                 from /root/project/include/K/IMAN4F_Switch.cpp:17:
/root/project/include/Qt5/QtGui/qopengl.h:141:13: fatal error: GL/gl.h: No such file or directory
 #   include <GL/gl.h>
             ^~~~~~~~~
compilation terminated.
CMakeFiles/iip_demo.dir/build.make:153: recipe for target 'CMakeFiles/iip_demo.dir/include/K/IMAN4F_Switch.cpp.o' failed
make[2]: *** [CMakeFiles/iip_demo.dir/include/K/IMAN4F_Switch.cpp.o] Error 1
CMakeFiles/Makefile2:72: recipe for target 'CMakeFiles/iip_demo.dir/all' failed
make[1]: *** [CMakeFiles/iip_demo.dir/all] Error 2
Makefile:83: recipe for target 'all' failed
make: *** [all] Error 2
Exited with code 2
```

일단 basic build 만 돌리고, docker에 openGL이랑 Python 설치해서 다시 업로드하자. 

HPX의 readme 가 rst라서 별생각없이 CircleCI달면서 rst로 바꿧는데 괜히 바꾼듯. md로 돌림


---


### 04.21 ~

https://github.com/kooBH/Qt5_practice/issues/6

---

ubuntu 포맷함. ubuntu에 기본적으로 OpenGL이 설치되어 있지 않았다. Qt의존성 에러 in CMake


### 04.29  
3일정도 Qt 의존성문제를 해결함, Spectrogram 다시 손보자.  
원래 쓰던 연습 프로젝트에 의존성 문제가 넘칠때니 그냥 여기서 가져와서 돌려버릴까

### 04.30
GCP 모듈/위젯 과 Spectogram을 그리는 모듈/위젯을 분리하자. 

통합은함. 이제 모듈이랑 연결하고. 전체적으로 어떤 방식,모양으로 작동할지 구상해보자.

일단 스펙트로그램은 커야할테니 다른 옵션 위젯들의 크기를 고정크기로 설정해야함. 
+ 고정 크기 위젯
+ 비례되는 스펙트로그램
+ 실시간 스펙트로그램

### 방향성
+ 큼직하게
+ compact하게 ->  margin 없이

Module Widget에서 폰트를 키웠더니 스위치하고 잘 안어울림. 그냥 모듈의 라벨을 스위치로 만들어버릴까. 켜지면 초록 꺼지면 빨강 이런 식으로?
Clickable Label을 만들어 봤는데 이게 훨씬 직관적인듯. 좀 싼티나지만 그건 다듬으면 될 사항. 

일단 Spec 부터 완성시키자. spectrogram module을 만들어 보자. 


### 05.01

링크문제 아직 유효함. 외부 라이브러리의 외부 라이브러리 링크를 건드리는 방법을 찾거나  
라이브러리들을 환경변수로 찾을 수 있는 위치에 인스톨 해줘야함

### 05.02  

#### spectrogram 에 추가되어야하는 항목
+ resize
+ repaint
+ real time
+ audio play
+ export

https://doc.qt.io/archives/qt-4.8/qscrollarea-members.html
QScrollArea 사용. 

#### 입력
+ wav를 받으면 그걸 fft 해야 하지 않는가. 
WAV,FFT,HANN 다 있어야함. 

+ file dialog && drag 다 지원하고 싶음

#### 이슈
+ Spectrogram 길이 or 크기. 다 메모리상에 담아 둘 수 있을 거 같은데, 그렇게 할까? 

---

레이아웃 설정. 스크롤. 파일 윈도우 추가함. Drag & Drop 하자

https://doc.qt.io/archives/qt-4.8/qdropevent.html
https://doc.qt.io/qt-5/qtwidgets-dialogs-findfiles-example.html
https://doc.qt.io/qt-5/dnd.html
https://doc.qt.io/qt-5/qmimedata.html#details

Drag & Drop 추가함.  

이제 wav 파일에서 spectrogram을 만드는 기능만 추가하면 됨. 
+ drag & drop 사용 방식을 좀 생각해보자. 

차라리 비어있는 위젯창에 드래그 드롭하면 새로은 스펙트로그램을 만들게 해야겠다


### 05.03

drap & drop 으로 KAnalysis widget에 추가 됨. 좀 더 깔끔하게 들어가게 하고 삭제버튼 같은 것도 추가해야겠다. 

### TODO 
+ fft  
+ 추가적인 버튼들
+ 다듬기  

이걸 하려면 위젯에 fft,hann, wav 를 다 달아야함. 어디다 달까? 독립적으로 작동시킬까? 어떻게 하지? 각각의 Spec 위젯에 주는 건 낭비일거 같지만서도 그렇게 까지 비용이 큰가 싶기도 하고. 

동작은  
1. wav 읽음

2. hann

3. fft

4. spec 그리기 

각 단계가 어디서 수행될것인가? 하나의 위젯에서? 부모 위젯에서? 각각의 spec 위젯에서? 

### 이슈
기존의 처리모듈들은 대체로 고정된 파라매터를 전제로함. 하지만 이거는 고정된 파라미터를 보장하지 않는다 ! 그냥 wav 넣으면 돌려야지
전반적인 구조 조정이 필요함

### 05.05

+ Spec 위젯만 resizable 하고 나머지 위젯은 fixed 로 하기
+ Spectogram만을 위한 fft 모듈
+ window 길이
+ overwrap size
+ sampleing rate

wav 파일만 받게함.  

process하는 모듈은 Analysis에서 가지고 있는걸로 해야함. 

기존의 process 클새스에 iterator을 명시하는 오버로딩 함수를 만들어서 쓰자.
hann 어차피 기본적으러 그렇게 구성되어 있었다.   
일단은 Num_FFT를 쓰자.  

Analysis에 모듈 다는 중

### 05.06

일단 spec의 크기를 wav파일에서 받아온 다음에 wav를 읽으면서  
원래 모듈이 작동하던 방식으로 img에 쌓은다음에 표시해야할듯. 

spec widget에 픽셀을 찍을 포인트를 계속 움직여가면서  
끝까지 그리면 표시하는 걸로.  

이게 크기가 딱 안 나눠질 수도 있음. 그럴경우에 읽을때 zero-padding을 해주는걸 고려해야함  
ceil(데이터 크기/shift_Size) 이 실제 spectogram의 길이가 될것이다


### 05.07

Sample 갯수  adjust 구문 추가.  
KSpectrogram 에 Update 하는 기능을 추가하자.  문제는 채널별로 만들어야함.. 각각의 spectrogram을 관리해야함. 
적당히 관리하게 하고  logspec을 하고 띄우는데 영 이상함

fft 와 hann 을 한 상태에서 nan  와 inf가 많이 있다. 원래 이런거 같지는 않은데.  
```C
        wav_buf->Convert2ShiftedArray(buffer);
```
에서 값이 안넘어간다. buffer가 nan 아니면 inf로 찍힌다.  

shift 가 잘 안되는듯. 버퍼의 후반부는 잘 찍힘. 

입력 규칙을 잘 따르지 않았음. raw와 data를 분리하지 않아서 생긴 문제.  해결함 .

원하는 색감이 나오게 튜닝해야함. 

LogSpec class 만듦.  

min max 기준으로 sigmoid로 하면 너무 쏠리는 경향이 있음. 분포를 다르게 하거나 아니면 colormap 자체를 
특정 구간에 강조를 주는 형태로 가야할듯. 

아. jet colormap  식 자체가 잘못된듯 
jet colormap 부터 새로 만들자

---

그리고 frame_size 고지곧대로 그리면 너무 크다. 사이즈 조절 해야할듯

---



### 05.08

#### TODO
+ colormap tuning
+ resizable
+ 닫기 버튼과 그에따른 위젯 관리
+ export(spec 별 export 와 export all)
+ parameter 설정 위젯
colarmap class를 만들자. 

colormap 이전에 튀는 값들을 어떻게 처리하는가가 중요함.  
(0,1)로 스케일링할 경우 대다수 값들이 0.5 이상이다. 몇백개에 하나씩 있는 0.1 때문에 spectrogram이 잘 찍히지 않음.  
audacitiy의 spectrogram을 참고하려는 중.  

이런 구문이 있다. 

```C
          // Take FFT
         RealFFT(windowSize, in.get(), out.get(), out2.get());
         // Compute power
         for (size_t i = 0; i < windowSize; i++)
            in[i] = (out[i] * out[i]) + (out2[i] * out2[i]);

         // Tolonen and Karjalainen recommend taking the cube root
         // of the power, instead of the square root

         for (size_t i = 0; i < windowSize; i++)
            in[i] = powf(in[i], 1.0f / 3.0f);

         // Take FFT
         RealFFT(windowSize, in.get(), out.get(), out2.get());
```

---

일단 logspec 했을 때 0 미만인 친구들은 버렸는데 예쁘게는 나옴. 
문제는 여기서 0미만인 친구들이 가지는 함의가 어느 정도 인가? 
Outlier를 쳐내는 좀 더 안전한 방법이 없을까?  
log spec한거를 min max 보정해서 (-3,3) 으로 한다음 tanh로 (0.1) 로 바꾼걸 칼라맵으로 매핑한게 가장 가시성은 있다.  

일단 Spectro는 이정도로 하고  좀 더 유틸적인 부분을 추가하자 .

+ scale 1,2,4 추가  

삭제와 export 를 추가하자. 

각 spec에 대한 버튼을 추가하려면 새로운 위젯 클래스 안에다 담아두는 형식으로 해야할듯.  

---

file:///opt/intel/documentation_2019/en/ps2019/getstart_clus.htm

--

### 05.09

콤보박스 코드 날려서 새로 짬. 
spec을 담는 specWidget 만듦. 사이즈 설정할때 안의 spec 사이즈 기준으로해서 사이즈가 약간 밀리는 듯. 마진을 어느 정도로 둘지 정해야겠다. 

close 를 하기 위해선 SpecWidget과 KAnalysis가 서로를 참조해야함.  

헤더 파일을 하나로 merge.  
선언과 호출은 달아둠.  정의 해야함.

### 05.10  

id 줄 필요없이 KSpecWidget의 pointer 로 가능할까? 

Close 구현함. id 없이 포인터로 처리.  
KAnalysis 관련된 클래스들에서 id 다 없앰.  

---

export 만듦. 
close all 만듦.  
1차 적인 건 된듯 이제 오디오 재생을 구현할 준비를 해보자.  

---

+ Spectogram 에 윈도우 크기 조절하는 걸 넣어야겠다 - 내장된 프로세서를 재생성하는 구문과 함께 - 
+ 파일명 표시하자  

### 05.13  

각각의 스펙트로그램 위젯에서 재생을 하려면 각각의 위젯이 자신의 채널에 대한 버퍼를 가지고 있어야한다.   

wav에서 Conver2ShiftedArray로 버퍼를 더블로 채널별로 쪼개서 받는데, 그걸 그냥 shift_size만큼만 short로 다시 돌려서 받아두면 될거 같다. 

```c
 for (int i = 0; i < channels ; i++){
             memcpy(buf_data[j][i], buf_raw[i], sizeof(double) * frame_size);
             short * temp_buf =  
              vector_spec.at(num_spec-channels+i)->Get_buffer_wav();
             for(int k=0;k<shift_size;k++)
               temp_buf[j*shift_size + k]=static_cast<short>(buf_raw[i][k+(3*frame_size)]);

```

대강 이렇게, 어차피 실시간을 요구하는 부분은 아니고 연산량자체도 적어서 이정도만 해도 괜찮을듯. 
마우스 포인터를 각 위젯에 대한 걸로 받을 수 있음.  
재생은 어느 위젯에서 관리할까? KAnalysis에서 Rt_Output을 관리해야할거 같은데. 

마우스 드래그로 도 추적함. 

https://doc.qt.io/qt-5/qwidget-members.html
https://doc.qt.io/qt-5/qdragmoveevent-members.html
https://doc.qt.io/qt-5/qwidget.html#mouseMoveEvent
https://doc.qt.io/qt-5/qmouseevent.html#pos

### 05.14

일단 드래그 영역표시까지만 하고 오디오 재생이 마무리 될 때까지 보류하자.  

+ 윈도우 변경 루틴을 추가해야함. 

드래그 영역 표시추가.    
윈도우 변경 콤보박스와 루틴 추가.  

---
 
문제는 libqxcb.so가 프로그램과 직접적인 연관이 없어서 런타임 환경을 설정하는게 아무 효과가 없는 것.  
cmake 시 path를 조작하는 작은 프로그램을 돌릴까?  
install_dependeny.sh 작성함.  

---

걍 스크립트로 하고 윈도우에서 dll이 같은 폴더에 위치한거를 검색한다면 괜찮을테니 일단 테스트부터 해보자. 
cmake 파일분할 해야겠다. 
cmake 모듈화해서 분리함.


### 05.15
  
Windows Dependency 해결하자. dll 3개 정도 추가함. 기타 Windows 버그도 해결.
windows에서는 라이브러리의 라이브러리 의존성 문제는 아직까진 발견 안됨.     


---

라이브러리 파일을 별도로 설치하는 루틴을 추가해야할듯. repo 크기가 너무 크다.   

https://curl.haxx.se/libcurl/   

---

비주얼 스튜디오 에러 발생. 내 노트북에서는 잘 돌아갔으나 서피스북에서 에러발생.  
같은 비주얼 스튜디오 2017인데 버전만 다를 것이다. 비주얼 스튜디오의 문제기는 하지만 이걸  
놔둘수도 없고, 어찌 해결은 해야할듯. 

### 05.16  

env 조작 테스트. setenv로 설정된 환경변수는 해당 프로그램의 런타임에서만 적용됨. 원하는 것은 프로그램에서 직접적으로 사용하지 않는 라이브러리의 PATH 설정이므로 사용하기 힘들것같다. 

---

ICC 돌려볼까.  
cmake 에서 icc를 잘 못잡는다. compilervars.sh를 해도 icc는 되는데 icpc를 못잡고있음.  

경로를 명시해서 돌리니까 ALSA 라이브러리를 못찾고 fail.  

설정하니까 무한 루프 돈다.  
CMakeLists.txt 내에서 set으로 하는건 지양하라고 하네. cmake command에 명시하라고 하고 스크립트 내에서는  
https://stackoverflow.com/questions/13054451/unable-to-specify-the-compiler-with-cmake
```
never try to set the compiler in the CMakeLists.txt file.
```

컴파일러를 확인하는 정도로만 사용한다.  
https://gitlab.kitware.com/cmake/community/wikis/FAQ#how-do-i-use-a-different-compiler 

.profile 에
```bash
source /opt/intel/compilers_and_libraries_2019.3.199/linux/bin/compilervars.sh -arch intel64 -platform linux
source /opt/intel/compilers_and_libraries_2019.3.199/linux/bin/iccvars.sh -arch intel64 -platform linux
```

추가하고

```cmake
cmake -DCMAKE_CXX_COMPILER=/opt/intel/compilers_and_libraries_2019.3.199/linux/bin/intel64/icpc  ..
```

구문으로 빌드 시도는 됨. C+11이 아닌걸로 빌드되서 에러. cmake 단에서는 명시를 했는데 컴파일 플래그로 넣어줘야할듯.  

일단 어거지로 빌드해서 돌린거는 별 차이가 없다. 빌드시간은 엄청 걸리지만. 플래그를 설정해야할듯. 


### 05.17

ICC 빌드. 
```cmake
function(GetNumOfProcessors num)
```

가 잘안되서 리턴이 없음.  

```
run Build Command:"/usr/bin/make" "cmTC_222b5/fast"
/usr/bin/make -f CMakeFiles/cmTC_222b5.dir/build.make CMakeFiles/cmTC_222b5.dir/build
make[1]: 디렉터리 '/home/kbh/git/IIP_Demo/build/CMakeFiles/CMakeTmp' 들어감
Building C object CMakeFiles/cmTC_222b5.dir/CheckFunctionExists.c.o
/usr/bin/cc    -DCHECK_FUNCTION_EXISTS=pthread_create   -o CMakeFiles/cmTC_222b5.dir/CheckFunctionExists.c.o   -c /usr/share/cmake-3.5/Modules/CheckFunctionExists.c
Linking C executable cmTC_222b5
/usr/bin/cmake -E cmake_link_script CMakeFiles/cmTC_222b5.dir/link.txt --verbose=1
/usr/bin/cc   -DCHECK_FUNCTION_EXISTS=pthread_create    CMakeFiles/cmTC_222b5.dir/CheckFunctionExists.c.o  -o cmTC_222b5 -lpthread
make[1]: 디렉터리 '/home/kbh/git/IIP_Demo/build/CMakeFiles/CMakeTmp' 

 /home/kbh/git/IIP_Demo/src/util/GetNumOfProcessors.cpp(2):
/usr/include/c++/5.4.0/bits/c++0x_warning.h(32): error: #error directive: This file requires compiler and library support for the ISO C++ 2011 standard. This support must be enabled with the -std=c++11 or -std=gnu++11 compiler options.
  #error This file requires compiler and library support \
   ^

/home/kbh/git/IIP_Demo/src/util/GetNumOfProcessors.cpp(6): error: name followed by "::" must be a class or namespace name
    unsigned concurentThreadsSupported = std::thread::hardware_concurrency()
                                              ^

compilation aborted for /home/kbh/git/IIP_Demo/src/util/GetNumOfProcessors.cpp (code 2)
CMakeFiles/cmTC_6bdc5.dir/build.make:65: 'CMakeFiles/cmTC_6bdc5.dir/GetNumOfProcessors.cpp.o' 타겟에 대한 명령이 실패했습니다
make[1]: *** [CMakeFiles/cmTC_6bdc5.dir/GetNumOfProcessors.cpp.o] 오류 2
make[1]: 디렉터리 '/home/kbh/git/IIP_Demo/src/util/CMakeFiles/CMakeTmp' 나감
Makefile:126: 'cmTC_6bdc5/fast' 타겟에 대한 명령이 실패했습니다
make: *** [cmTC_6bdc5/fast] 오류 2
]
```

c++11 이 여기서 명시가 안되었네. cmake try_run  에는 cmake flag로 밖에 옵션을 줄 수가 없는데 ,  
ICC 를 사용할 떄는 cmake 옵션이 잘 안먹는다. 되는 옵션을 찾아보자.  

아니면 c 코드로 프로세서 수를 받아 올까?  
C 코드로 바꿈.

---

https://software.intel.com/en-us/articles/step-by-step-optimizing-with-intel-c-compiler

일단은

```
-std=c++11
-openmp
-parallel         #enable automatic parallelization
-O3
#   -ffast-math
#-fast
-ansi-alias
-ipo              #  enable interprocedural optimizations across source files,
                  #  including inlining
-guide            #  enable guided automatic parallelization and
                  #  vectorization; compiler gives advices on how to get
                  #  loops to vectorize 

	    -march=native

```

이렇게 넣어 봤는데 별다른 차이가 없다. wpe 만 돌려서 그런거 같은데, 전반적인 테스트 방식을 생각해보자. 
옵션적용이 안되었었다. 옵션넣어서 빌드하려니 

```

TestModule.cpp:(.text+0x309f): undefined reference to `__kmpc_for_static_init_4'
TestModule.cpp:(.text+0x32a2): undefined reference to `__kmpc_for_static_fini'
TestModule.cpp:(.text+0x3373): undefined reference to `__kmpc_for_static_init_4'
TestModule.cpp:(.text+0x33c9): undefined reference to `__kmpc_for_static_fini'
TestModule.cpp:(.text+0x3410): undefined reference to `__kmpc_global_thread_num'
TestModule.cpp:(.text+0x3452): undefined reference to `__kmpc_global_thread_num'
TestModule.cpp:(.text+0x3670): undefined reference to `__kmpc_ok_to_fork'
TestModule.cpp:(.text+0x369a): undefined reference to `__kmpc_fork_call'
TestModule.cpp:(.text+0x36af): undefined reference to `__kmpc_serialized_parallel'
TestModule.cpp:(.text+0x36e0): undefined reference to `__kmpc_end_serialized_parallel'
TestModule.cpp:(.text+0x36ec): undefined reference to `__kmpc_ok_to_fork'
TestModule.cpp:(.text+0x3717): undefined reference to `__kmpc_fork_call'
TestModule.cpp:(.text+0x372c): undefined reference to `__kmpc_serialized_parallel'
TestModule.cpp:(.text+0x375e): undefined reference to `__kmpc_end_serialized_parallel'
TestModule.cpp:(.text+0x37ae): undefined reference to `__kmpc_ok_to_fork'
TestModule.cpp:(.text+0x37e5): undefined reference to `__kmpc_fork_call'
TestModule.cpp:(.text+0x37fe): undefined reference to `__kmpc_serialized_parallel'
TestModule.cpp:(.text+0x3847): undefined reference to `__kmpc_end_serialized_parallel'
TestModule.cpp:(.text+0x38f9): undefined reference to `__kmpc_ok_to_fork'
TestModule.cpp:(.text+0x3930): undefined reference to `__kmpc_fork_call'
TestModule.cpp:(.text+0x3949): undefined reference to `__kmpc_serialized_parallel'
TestModule.cpp:(.text+0x3992): undefined reference to `__kmpc_end_serialized_parallel'
TestModule.cpp:(.text+0x3e9b): undefined reference to `__kmpc_global_thread_num'
TestModule.cpp:(.text+0x3eca): undefined reference to `__kmpc_global_thread_num'
TestModule.cpp:(.text+0x3f63): undefined reference to `__kmpc_global_thread_num'
TestModule.cpp:(.text+0x3f86): undefined reference to `__kmpc_global_thread_num'
TestModule.cpp:(.text+0x3f97): undefined reference to `__kmpc_global_thread_num']
```

이런 에러 발생

https://software.intel.com/en-us/forums/intel-fortran-compiler-for-linux-and-mac-os-x/topic/267525

여러가지 옵션들 시도는 했는 데 잘안되네

---

SRP 에서 프로세서가 30개가 넘을 경우를 처리하지 않음. circleCI 서버가 36프로세서임.  

```C
#define OMP_SRP ((30/_NUM_OF_PROC)>1?(30/_NUM_OF_PROC):1)
```  
circleCI에서 이제 에러 안남.

### 05.20

icc build 에러

```
/usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/crt1.o: In function `_start':
(.text+0x20): undefined reference to `main'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x0): undefined reference to `__sti__$E'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x8): undefined reference to `__sti__$E.4'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x10): undefined reference to `__sti__$E.5'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x18): undefined reference to `__sti__$E.6'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x20): undefined reference to `__sti__$E.7'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x28): undefined reference to `__sti__$E.8'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x30): undefined reference to `__sti__$E.12'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x38): undefined reference to `__sti__$E.13'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x40): undefined reference to `__sti__$E.14'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x48): undefined reference to `__sti__$E.15'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x50): undefined reference to `__sti__$E.16'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x58): undefined reference to `__sti__$E.17'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x60): undefined reference to `__sti__$E.18'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x68): undefined reference to `__sti__$E.19'
/tmp/ipo_icpcPBfS1w.o:(.ctors+0x70): undefined reference to `__sti__$E.20'
/tmp/ipo_icpcPBfS1w.o:(.rodata._ZTVN6serial6SerialE[_ZTVN6serial6SerialE]+0x10): undefined reference to `serial::Serial::~Serial()'
/tmp/ipo_icpcPBfS1w.o:(.rodata._ZTVN6serial6SerialE[_ZTVN6serial6SerialE]+0x18): undefined reference to `serial::Serial::~Serial()'
/tmp/ipo_icpcPBfS1w.o:(.rodata._ZTVN6serial6Serial10SerialImplE[_ZTVN6serial6Serial10SerialImplE]+0x10): undefined reference to `serial::Serial::SerialImpl::~SerialImpl()'
/tmp/ipo_icpcPBfS1w.o:(.rodata._ZTVN6serial6Serial10SerialImplE[_ZTVN6serial6Serial10SerialImplE]+0x18): undefined reference to `serial::Serial::SerialImpl::~SerialImpl()'
/tmp/ipo_icpcPBfS1w.o:(.rodata._ZTV8RT_Input[_ZTV8RT_Input]+0x20): undefined reference to `RT_Input::CleanUp()'
/tmp/ipo_icpcPBfS1w.o:(.rodata._ZTV6Module[_ZTV6Module]+0x18): undefined reference to `Module::~Module()'
/tmp/ipo_icpcPBfS1w.o:(.rodata._ZTV6Module[_ZTV6Module]+0x20): undefined reference to `Module::~Module()'
/tmp/ipo_icpcPBfS1w.o:(.rodata._ZTV6Module[_ZTV6Module]+0x28): undefined reference to `Module::Stop()'
/tmp/ipo_icpcPBfS1w.o:(.rodata._ZTV5RtApi[_ZTV5RtApi]+0x10): undefined reference to `RtApi::~RtApi()'
```

https://software.intel.com/en-us/forums/intel-c-compiler/topic/289668  
-parallel 을 link 옵션으로 옮겼더니 에러가 좀 줄긴함. 아니면 옵션이 안들어갔거나.  

일단  
c++11, openmp, O3, arch=native,lpthread 로는 빌드 성공

+ parallel 을 컴파일 옵션으로 두면 __kmpc_ 관련 에러 발생, 링크 옵션에 둠.
+ anis-alias 는 괜찮음 
+ guide 옵션에서 에러 발생. 

아직까진 성능 향상이 없다. Advsior 써보자

---

anis-alias 를 좀 알아보자 

https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.3.0/com.ibm.zos.v2r3.cbcpx01/optalias.htm  
타입 캐스팅에 더 엄격한 룰을 적용시켜서 컴파일러가 더 공격적으로 최적화를 수행하게 함.  

---

advisor 돌리는 스크립트 추가함.
advisor 를 공부해보자

---

AEC가 그냥 안되는 거였다.

일단 결과가 가 0으로 뜨고. 결과가 다 0일때 min max도 다 0이라서 Qt에서 에러 발생.  예외처리부터 달고 고쳐보자...  

```C
const double gap_min = 10;
```

예외처리함. 

언제서부터 안된거지? 간간히 check를 돌려서 오처 0.0000 나오는거 확인하고 있었는데, aec 코드를 바꾼적은 없으니까.   
다른 거에서 변경된게 영향을 미치는거 같다.

잘돌아가가는 레퍼런스 프로젝트와 비교해보자 ..

SIMD 플래그가 컴파일 플래그 설정된 다음에 설정되었었다. 

cmake 코드 수정함. 

---

### 05.21

워닝들 좀 없에고 플래그 수정. 

---
업 샘플링 라이브러리 부분 master로 pull.
NOTICE 추가하고 README 에 외부 의존성 추가함. 

---

재생을 위해 재생영역선택 후 재생시 선택영역의 버퍼를 만드는 기능을 구현하자. 일단은 단일 채널에 대해서만 구현을 하자. 
두가지 영역 지정 방법이 있다. 
1. 시작점만 지정 -> 뒷 부분 전부 재생
2. 영역 지정 -> 지정된 영역만 재생
3. 더블 클릭시 전체 영역

마우스 드래그 상태로 분기해서 조건을 설정하면 될듯. 

영역지정을 어떻게 유지하는가.

영역지정하는 구문 추가. 
```cpp
    /* Audio Play */
    size_t offset_start,offset_end;
    size_t offset_play;
   inline void set_offset(size_t start,size_t end);

```

난감한 것이. 
KAnalysis 안에 KSpecWidget 안에 KSpectrogram 이 있는데, 마우스 입력을 KSpectrogram에서 받는다. 스펙트로그램 자체에서만 
이식을 하면 편해서 문제는 이렇게하면 여러 Spectrogram을 같이 영역지정을 하기가 힘들어지고 전체적인 관리를 하기가 힘들다. 2단계의 콜백으로 KAnalysis와 주고받을 수는 있지만..  

구성을 다시 해야할까???  이 상태면 오디오 재생의 함수들은 2단계의 콜백들로 이루어질것인데, 기능상의 문제는 없겠지만 모양이  
마음에 들지않는다. 하지만 음 .. 일단 단일 채널에 대해서만 재생한다 치면 2단계가 아닌 1단계의 콜백으로 할 수 있으니... 하지만 역시.. 다 채널 재생을 해야겠지? 

다른 건 다 구현할 수 있는데, 여러 스펙트로그램의 동시 영역지정은 좀 생각을 해야겠다. 마우스 입력관련해서 다시 짜야할수도 있음.

많이 무식한 방법인데, 하나의 스펙트로에서 드래그 모드임이 인지되면 다른 스펙트로그램들도 다 드래그 모드로 해버리는 식의 전체 스펙트로그램의 마우스 입력 이벤트 들을 관리하는 루틴의 Analysis 웨젯에 추가하면 어찌 될지도 모른다.  

사실 시작 지점의 인덱스도 공유해야해서 이럴 수 밖에 없다고 생각함. 지금은.  


---  

### 05.22  

일단 위젯의 리스너가 별도의 스레드에서 돌아가는 지 알아보자. 안됨.  별도의 쓰레드를 만들어서 돌려야한다. 쓰레드..  

두가지 선택지.
1. 재생중에는 해당 위젯의 마우스 영역지정을 못하게 한다.
2. 재생중에 마우스 영역지정을 가능하게 한다. 

오다시티도 돌려보니 재생 지침이 도는 동안 마우스 영역지정을 할 수 있지만, 재생 지침이 버벅거리게 됨. 렉이 약간 걸린다. 마우스 영역지정 자체가 재생을 위한 준비단계이니 재생중에는 마우스 영역지정을 금지하자.   

wait 하는게 아니라 그냥 금지 시키는 것이니까 플래그로 하자. 

한번에 여러개를 플레이시 에러 발생. 
https://www.qtcentre.org/threads/68276-Getting-a-SIGSEGV-error-when-loading-a-pixmap  

QPixmap은 하나의 글로벌 캐시를 사용한다고함. 바이너리를 링크해서 사용해서 자세한 코드를 볼 수는 없지만 일단은 여러개의 쓰레드로 여러 픽스맵들을 연산할 때 생긴 문제이니 이게 가능성이 커보인다. 

```cpp
QPixmap::fromImage() 
```
가 문제일까?  

https://doc.qt.io/qt-5/qpixmap.html#convertFromImage

```cpp
bool QPixmap::convertFromImage(const QImage &image, Qt::ImageConversionFlags flags = Qt::AutoColor)
```
이걸 써보자 

해결됨. 각각의 QPixmap buf를 사용해서 글로벌 캐시를 사용하지 않게 다 바꾸자. 바꿈.  

여러개 재생 시 몇몇 위젯의 업데이트가 멈춘다. 

이것도 비슷하게 GUI를 담당하는 쓰레드가 별도로 있다면 프리징 현상이 말이 되긴한다. 

pixmap copy에서 에러가 나는데 img -> pixmap을 pixmap 자체에 그리는게 처음에 번거로워질거 같아서  
그랬는데 이제  img 부분을 빼고 pixmap으로만 할 수 있는지 봐야겠다. 

DisplayTest에서 img를 갱신해서 사용하지 않게 하였으니 잘 되지 않는다.  

UI는 메인쓰레드에서만 다룰수 있다는 글이 있는데.. QThread를 써야하는가. 

구조를 바꾸보자. Analysis에서 하나의 쓰레드만 추가되어서 spec들에게 업데이트를 시키는 걸로.  

하나로 하는건 잘 돌아가는 듯. 하지만 완전한 해결은 아니다.
더욱이 cpu 리소스를 너무 많이 먹는다. 오다시티보다 5배는 먹는듯. cpu 하나를 거의 80% 까지 사용한다. 

일단 중간 정리 먼저.

테스트 도중 한번 멈추는 상황이 발생, update를 호출하는 걸 임의로 조작하면 update되는 걸 보아서. 기존의 문제가  
그대로인거 같다.  

---

일단 단일 채널에 대해서만 재생을 추가해보자. 
+ play 시작 시
1. 선택된 영역의 범위를 받는다.
2. 시작 부분부터 버퍼를 shift_size 만큼 재생 시킨다.
3. 재생 속도에 맞게 indicatior를 움직인다.


---

scale이 8일때 버튼 다 짜부러진다. 

---

### 05.23

stop 과 stop 시 사용될 spectrogram 의 refresh() 추가함. 

이제 영역지정을 고려하자. 
mouseMoveEvent를 여러 위젯에 걸쳐서 시그널을 보낼 수 있을거라 생각했는데 말이 안되네. 어디서부터 어디까지 지정되었는지
결국엔 포인터 좌표로 처리해야하니까. 
 
Analysis에 에서 마우스 포지션 받아서 대응하는 spec을 각각 계산해야하고 그리라는 시그널을 보내줘야겠네. 문제는 이게 스크롤까지 생각하면 좀 많이 복잡해진다.

어쩔수 없이 Analysis에서 포인터를 다뤄야할듯. 

Analysis에 SpecWidget이 있을 경우 SpecWidget으로 시그널이 들어감. spec부분은 spec에 들어가겠지 ..  
전체에서 마우스 입력을 아루르는 방법이 없을까? 

spec에서 받으면 다른 analysis로 올려서 거기서 처리해서 뿌릴까? 
+ top-down?
+ bottom-up&down?

재생은 어차피 top에서 해야하고.. 그거 고려해서 정하자. 

두번째 껄로하자. 어차피 Analysis로 해도 이벤트 필터를 달아야하는 거 같으니까.구조의 차이는 없을듯.  

1. 기준이 될 채널의 인덱스 클릭
2. 드래그
3. 드래그의 y값이 다른 채널에 포함되어있다면 해당 채널도 영역지정에 포함되어 있는지 확인하여 포함
4. 계속 추적하면서 채널의 포함 유무를 갱신
+ x 값 영역도 계속 추적



---

SpecWidget 마진들 다 최소화함. 

### 05.24

추가적인 예외처리함. 

영역지정의 알고리즘은 스크롤을 생각해야한다. 여러가지를 시도해봐야할듯.  
일단 각 위젯에게 id를 부여하자. 

mapToGlobal 과 mapFromGlobal 을 잘 쓰면 쉽게 될거 같다.  

좌표계가 생각한거랑 좀 다르다. 아니면 윈도우를 움직여서 그런 걸 수도? 

기능 구현은 함.  이제 정리하고 코멘트를 달자.  그 다음에 버퍼 관리해서 재생기능 추가하자. 

클릭시 Refresh 도 해야한다. 

----

### 05.25 

위에 시간표시하는 것도 달아야겠다. 
윈도우도 512/2048 추가함.  
widget_west 가로 크기 고정.  

정리랑 클릭시 refresh 해야함.   

버퍼 범위를 signal & slot 으로 받아보자. 틀은 구현함.


---

### 05.26

동적으로  signal을 slot에 다는게 안되는건가? 

https://doc.qt.io/archives/qq/qq16-dynamicqobject.html

**컴파일**시 매크로 전처리로 연결함, 
```
normally you cannot add new signals and slots to a meta-object at run-time.
```

normally 니까 방법이 있다는것. 이건 2005년 Qt4 문서. 10년도 넘었다. 

https://stackoverflow.com/questions/45265391/how-to-connect-in-qt-signal-and-slot-in-dynamically-added-buttons-to-get-in-slot

### 05.27  

저거 알아볼 시간에 함수달아서 돌리는게 빠를듯. 사실 실제로 돌아가는 메커니즘도 별 차이 없을 거다. 

RefershSpec추가함. 작동확인.  

1. 이제 local 에 따른 버퍼 크기를 Ana에 보내기.
2. play시 선택된 영역의 버퍼를 wav 포맷으로 합치기.
4. 선택된 영역 내에서 실시간으로 재생위치 표시하기
3. 합친 wav 버퍼를 실제로 재생하기.
4. 일시정지
5. 중지

를 구현하자. 

SetArea는 ana에서 해당되는 spec에 뿌려서 돌리는 함수. 지금 얻고자하는건 Ana 단에서 사용될 범위.  
SeaArea와 별도로 함수를 추가해보자. 

범위 벗어났을 때의 예외처리 해줘야함. 

---

Mac에서 빌드 안됨. 일단 openmp부터 안된다. 

https://stackoverflow.com/questions/35134681/installing-openmp-on-mac-os-x-10-11

brew install clang-omp 는 안된다. 
boneyard 가 deprecated 되었다고 하는데?  


https://stackoverflow.com/questions/38971394/brew-install-clang-omp-not-working

```
brew install llvm
```

시간이 많이 걸린다. 20분 넘게 걸린다. make 돌리니까 맥북이 무시무시한 소리를 낸다. 
11시 50분 부터 돌림. 

```
brew install libomp
```

하고 링크,인클루두 걸어주니까 omp 빌드는 되는다 심볼릭 링크가 다 깨짐.  std가 안된다. 
이건 c++ 을 clang 해서 그럼 clang++로 하니 해결.  

openmp에서 KMPC 관련 에러 난다. 
alinged_malloc 도 에러 

c++11 이상이 지정안되서인데. 플래그로 줘야함.  
지금은 c,c++ 플래그 구분이 안된상태라 clang에서는 c에 들어간 std=c++11 이 에러표시.  

https://stackoverflow.com/questions/38578801/target-compile-options-for-only-c-files

https://cmake.org/cmake/help/v3.13/manual/cmake-compile-features.7.html

일단 그냥 다른 c++ 코드랑 섞여서 들어가는 c 코드들 다 cpp로 돌렸음. 

alinged_malloc은 c++11인데 clang은 없다고한다. g++를 설치해야하나


---

### 05.25

algined malloc 이 없는게 아니라 전처리에서 안들어간게 아닌가? 
ㅇㅇ 맞음. 해결함.

---

_omp_get_thread_num 해결하자. 

https://stackoverflow.com/questions/43555410/enable-openmp-support-in-clang-in-mac-os-x-sierra-mojave


---

MAC Qt5

https://www.qtcentre.org/threads/11124-Using-so-libraries-under-Mac  
Short answer: you cannot use libraries compiled for OS Foo with OS Bar. So, no, you cannot reuse Linux libraries for MacOS.  

OSX용 Qt5 설치하고 라이브러리 추출해야함. 

---

RtAudio OSX용 파일들을 안 넣었나? 플래그 설정도안해둠. 
설정함. API - Core 확인함.  

노트북 디폴트 마이크 1채널짜리 있길래 써보려했는데 seg 발생. 한번 알아보자. 

### 05.29

+ OSX의 호환성은 일단 재생 부분만. -> 문서작성해야함.
+ CLI 를 현재 config 방식에 맞게 수정해야한다. 

호환성 부분 merge 했고 cmake 조금 다듬고 문서작성하자. 
관련 문서 작성함. 

---

### 05.30

audio 재생 UI 계속하자. 

1. 이제 local 에 따른 버퍼 크기를 Ana에 보내기.
2. play시 선택된 영역의 버퍼를 wav 데이터 순서로 합치기.
4. 선택된 영역 내에서 실시간으로 재생위치 표시하기
3. 합친 wav 버퍼를 실제로 재생하기.
4. 일시정지
5. 중지

1 까지는 구현함. 버퍼합치는 부분을 만들자.
  
range * 선택된 영역 * shift_size 의 버퍼에다가 채워넣으면 될듯함.  
길이가 다른 버퍼를 같이 선택하였을 시의 예외처리를 생각해야함.    
버퍼 합치는건 구현했는데 돌려봐야 잘 되었는지 알 수 있을듯.  

재생하는거 sh 오면 하자. 좀 다듬고 인터페이스도고치고 해야할듯. 

선택된 영역 내에서 실시간으로 재생위치 표시하기 할 준비는함. 버벅거리는 현상 발생.  

---

icc 관련  cmake 정리하자.  

verdigris가 ICC에서  빌드 에러. 안되는거 같음. 

---



### 05.31

표시할때 repaint가 실시간으로 안되는걸 해결자하자. 
일단 update를 넣어둠.  재생중에 마우스 이벤트가 발생하면 위젯 업데이트가 깨진다.  
```releaseMouse()```랑 ```grabMouse()```를 쓰려 했는데  
https://doc.qt.io/qt-5.9/qwidget.html#grabMouse  
```Warning: Bugs in mouse-grabbing applications ``` grab이 잘 안된다.   
```grabMouse()```자체에 리스크가 큰듯  

아.모든 라벨에 ```SetDrawable(false)```를 해줘야지.  

많은 라벨을 돌리면 재생이 안됨. 근데 그렇게 많은 채널을 재생하는게 애초에 안되기는 하는데..

디버그 버튼 달아서 확인하니까 Drawable은 문제가 없는데 드래그가 잘 않잡히는 걸 보면  
트래킹 쪽 플래그가 잘 안 다뤄진듯하다. 

https://stackoverflow.com/questions/6420917/using-qwidgetupdate-from-non-gui-thread  

https://doc.qt.io/qt-5.9/qwidget.html#paintEvent
```
Note: Generally, you should refrain from calling update() or repaint() inside a paintEvent(). For example, calling update() or repaint() on children inside a paintEvent() results in undefined behavior; the child may or may not get a paint event.
```

https://stackoverflow.com/questions/28698904/qt-the-correct-way-to-display-a-large-amount-of-real-time-data-as-text

spectrogram을 그리는 함수를 커스텀하게 짜야하는건가?   

https://forum.qt.io/topic/86500/understanding-paintevent/4  

https://doc.qt.io/qt-5/graphicsview.html 로 대체해볼까?   

일반적인 상황에서는 잘 작동하니까 보류? 
특수상황 처리를 따로 이슈로 하자.

----

install_configuration 다듬고 ```README.md``` 업데이트함.  

### 06.03

CLI 수정 중. PARAM은 VALUE를 직접 넣는게 아니라 OPTION중에서 고르는 형식이라 별도의 루틴을 만들어야함.  
json  리스트는 std::list로 받을 수 있는데,  

https://en.cppreference.com/w/cpp/container/list  
 ```Fast random access is not supported```..  

CLI PARAM 부분 수정완료. 동작 확인.  

---

VS 에서 Clang 사용하는게 어느정도 난이도인지 파악해보자.  
VS2017 64bit, win10.  

도구 -> 도구 및 기능 가져오기.  
설치관리자 업데이트  
는 아닌거 같다.  

확장 및 업데이트 -> 온라인 -> clang 검색.  

1. [Clang Power Tools](https://marketplace.visualstudio.com/items?itemName=caphyon.ClangPowerTools)
2. [LLVM Compiler Toolchain](https://marketplace.visualstudio.com/items?itemName=LLVMExtensions.llvm-toolchain)  

둘 다 LLVM을 설치하고 그거를 연결시키는 툴.  

알기로는 확장 프로그램이 아니라 기본 VS 설치관리자에서 선택할 수 있다는거 같은데.  
https://stackoverflow.com/questions/43464856/integrate-llvm-clang-4-x-x-5-x-x-6-x-x-into-visual-studio-2017   
VS 2015만 된다고?  모바일 항목에?  

clang power tools 는 clang 컴파일러라기 보다는 clang-format 과 tidy같은 코드 변환에 중점을 둔거 같다.   
https://github.com/CppCon/CppCon2018/raw/master/Posters/retrofit_cpp17_to_large_visual_studio_codebases/retrofit_cpp17_to_large_visual_studio_codebases__gabriel_diaconita__cppcon_2018.pdf

ICC 도 덜 했는데 CLANG까지 건드리는 건 시기상조라 생각.  일단은 여기까지.  

---


Win10 64bit, VS2017 ```wobjectimpl.h```
```cpp
 constexpr auto sigState = w_SignalState(w_number<>{}, static_cast<T**>(nullptr));
    constexpr auto methodInfo = binary::tree_cat(sigState, w_SlotState(w_number<>{}, static_cast<T**>(nullptr)),
                                                    w_MethodState(w_number<>{}, static_cast<T**>(nullptr)));
```
```심각도	코드	설명	프로젝트	파일	줄	비표시 오류(Suppression) 상태
오류	C2440	'초기화 중': 'void (__cdecl *)(bool)'에서 'F'(으)로 변환할 수 없습니다.	iip_demo	c:\projects\cpp\iip_demo\libs\verdigris\wobjectimpl.h	199	
```

https://github.com/woboq/verdigris/issues/44  
VS 컴파일 에러라고 함.  VS를 업데이트 하고 돌려보자. 빌드 성공. 업데이트가 중요하다.


---

일단 기본상태에서 advisor를 사용해보자.

Memory Access Patterns analysis.  

컴파일러가 긴가민가해서 아무것도 안하는 부분에 대해서 ```#pragma```로 명시를 해줄 수 가 있는데,  
이게 ICC 전용이라 코드를
```cpp
#ifdef __ICC__
#pragma ivdep
#endif
```
로 할지 그냥 워닝 띄우면서
```cpp
#pragma ivdep
```
로 할지.. 일단 코드가 너무 길어지는건 안 좋으니까 그냥 pragma 하려해도, 명시가 안되어있으면 헷갈릴 수도 있기 때문에 생각을 좀 해봐야겠다.  

---

OSX serial port

애초에 인식자체를 못한다. 뭐가 문제일까. 

https://discussions.apple.com/thread/8560971  

하드웨어 문제인지 소프트웨어문제인지 부터 알아봐야한다. 현재 상태로는  
http://wjwwood.io/serial/ 여기에 해당하는 이슈도 없고 그냥 usb 인식이 안되는다는 거는,  
검색결과가 너무 광범위하기 떄문에 차근차근 가능한 원인들을 소거해가면서 파악해야겠다.  


-----

### 06.04 

재생되는거 확인함. 다만 upsampling 단위가 충분하지 않으면 노이즈가 발생. 32768정도는 되야 노이즈가 발생하지 않는다.  
Rt_Output의 전체적인 인터페이스를 수정해야하며, 정리를 해야겠다.  
재생 부에 정지를 - 일시 정지는 필요한가? -  추가하고 화면표시와 동기화할 수 있게 해야한다.  그 다음에 spectrogram 을 다루는 부분을 재설계해보자.  

여러 채널일 때 이상함. 스테레오가 스테레오가 아니라 섞여서 나온다. 

```cpp
  SRC_DATA src_data;
  memset (&src_data, 0, sizeof(src_data));
  src_data.src_ratio = oData->sample_rate/16000.0; //upsampel ratio for 16000 -> 44100
  //std::cout<<"src_ratio"<<src_data.src_ratio<<std::endl;
  src_data.input_
  src_short_to_float_array((const short*)testbuffer, (float*)float_buffer, oData->channels*nBufferFrames);
  
  //resampling

  src_data.data_in = (float*)float_buffer;
  src_data.output_frames = (int)oData->channels*nBufferFrames;
  src_data.data_out = (float*)calloc(src_data.output_frames,sizeof(float));


```

upsampling 하는 부분을 보면 채널에 대한 명시가 없음, 그냥 통채로 생플링 해서 섞이는듯. 그렇다면 채널별로  리샘플링을 한다음에 합쳐야할듯. 

```cpp
short **buf;
buf에 short[채널][길이]; //할당하고 채워 넣음
RT_Output rt(...);
rt.load(buf,채널,길이);
rt.Start();
```

이런 식으로  받은 뒤에

```cpp
for i in 채널:
    buf[i] 을 리샘플링;
for i in 길이:
   for j in 채널:
      buf_wav[i*j + j] = buf[j][i] ;
```

이렇게 해서 buf_wav를 재생

---

indicator에서 딜레이는 samplerate/shift_size 인데, samplerate는 16K로 고정한다고 생각하자.  
각 shift_size 에 맞는 딜레이 구현함. 
 
---  

재생 도중에 사용되지 않는 버튼들 disable 해줘야한다.  

---

KSpectrogram에 paint를 동적이 아닌 정적으로 해둬보자. 

---

CLI 프로그램, 최대한 간단하게, 어차피 대부분은 구현이 되어있다. 그러면 녹음하는 부분만 별도로 컨트롤 하는 구문을 추가해서 만들면 될거 같다.  
컨트롤 만 받는 thread를 별도로 생성해서 해주면 될듯. 키 입력을 통해서 시작과 정지를 통제하는 것이니, 일단 ```sub_record```를 베이스로 만들자. 

아 생각보다 훨씬 프로그램이 많이 엮여 있네.  

cmake 단에서 그냥 프로그램 자체를 새로 만들기로함. 다 짰는데, 다중 정의 에러가 나네... 아이고. 
이런 에러 날만큼 코드가 길지도 않은데.. 

간단하게 구축해서 일단은 전송, 미흡한 부분이 많지만 피드백을 받아서 그쪽으로 수정해 나가는 것으로. 

---


### 06.05

https://github.com/kooBH/IIP_Demo/issues/206  

```libqxcb.so``` 해결 libs가 아니라 lib에 두고 Qt5 폴더를 따로 두지 않음으로 해결.  이런걸 명시를 안해주니까 힘들다.  

---

### 06.07

간단한 blas 성능 test를 해보자. 실제로 있는지 파악부터 하고. 


WPE의

```cpp
                    t_re = 0.0;
                    t_im = 0.0;

                    for (j = 0; j < NT; ++j) {
                        t_re += g[real][j * Nch + i] * X_delay[real][j] +
                            g[imag][j * Nch + i] * X_delay[imag][j];
                        t_im += -g[imag][j * Nch + i] * X_delay[real][j] +
                            g[real][j * Nch + i] * X_delay[imag][j];
                    }
                    buf[i][real] = X[i][real] - t_re;
                    buf[i][imag] = X[i][imag] - t_im;
```
는

```
g[Nfreq*2][Nch*tap*Nch];
X_delay[Nfreq*2][Nch*tap];
```

일 때, 각각의 복수수에 대해서 [CH][NT] * [NT] = [CH] 인 행렬 곱이다. 행렬곱이긴 한데 주 범위가 프레임이 아닌 채널과 탭 크기라 어느정도까지 빨라질지는 돌려봐야알듯. 

blas부터 설치하자..  github에서 바로 받는건 develop 버전을 받기 때문에 ```make```시 에러가 발생해서 stable한거 압출풀어서 사용하기로함. 빌드 성공. 테스트 프로그램 만들자. 

https://github.com/gogyzzz/iip_sph_pp/blob/master/source/iip_blas_lv3.c  

colMajor, rowMajor..이거 현재 코드가 어떻게 되는가에 따라 일이 두배가 될 수 도 있겠다. 일단 colMajor 인거 같음. 

```

/*
 **  cblas_?gemm(layout,transA,transB,m,n,k,alpha,A,lda,B,ldb,beta,C,ldc)
 **
 **    layout :   i) --->CblasRowMajor
 **    			   [0][1]  =  {0,1,2,3}
 **       	   [2][3]
 **
 **             ii)  |  [0][2] = {0,1,2,3}
 **                  |  [1][3]
 **                 \_/ CblasColMajor
 **
 **
 **   C := alpha * op(A)*op(B) + beta*C
 **
 **     op(X) =  	  i) X      when transX = CblasNoTrans
 **
 **     		 	 ii) X**T     ''        = CblasTrans
 **
 **     			iii) X**H     ''        = CblasConjTrans
 **
 **      m  = the number of rows of op(A)
 **      n  = the number of columns of op(B) and C
 **      k  = the number of columns of op(A) and rows of op(B)
 **
 **
 **      lda : the first dimension of A
 **      ldb : 		''	     B
 **      ldc :		''	     C
 **
 ** 		-the first dimension : the number of columns when CblasRowMajor
 **		                 	   	''    rows   when CblasColMajor
*/

```

```cpp

void test_omp(double**g,double**X_delay,double**x){
double t_re = 0.0;
double t_im = 0.0;
int i,j,k;
#pragma omp parallel for schedule(static,8) private(i,j,t_re,t_im)
  for (k = 0; k < Nfreq; k++){
    const int real = 2 * k;
    const int imag = 2 * k + 1;
      for (i = 0; i < Nch; ++i) {
        t_re = 0.0;
        t_im = 0.0;
        for (j = 0; j < NT; ++j) {
          t_re += g[real][j * Nch + i] * X_delay[real][j] +
            g[imag][j * Nch + i] * X_delay[imag][j];
          t_im += -g[imag][j * Nch + i] * X_delay[real][j] +
            g[real][j * Nch + i] * X_delay[imag][j];
        }
      }
  }
}
void test_blas(double**a,double**b,double**c){
  CMP alpha;
#pragma omp parallel for
  for(int i=0;i<Nfreq;i++)
cblas_zgemm(CblasColMajor,CblasNoTrans,CblasNoTrans,
    Nch,1,NT,&alpha,a[i],Nch,b[i],NT,&alpha,c[i],Nch);
}

```
iteration 2만  
8.3초 vs 1.5초..

다르게 해서 돌려도 최소 4배는 빠르다. 

쓰레딩 연산 대상이 작기 때문에 blas의 쓰레딩은 끄고 omp로 프레임을 돌리자. 저건 테스트 값이 의뭉스러움. 

오차가 너무나는데, 뭘 잘못 넣은 걸까. 

아 데이터 구조가 문제다.  re im re im 있어야하는데, 원래 데이터 구조가 re[CH * NT] im[CH * NT] .. 와.. 
blas 돌리려면 double double 구조체로 [Nfreq][CH * NT]로 만들던가 double 로 [NFreq][CH * NT * 2]로 해야하네, 

별도의 구조체를 사용하면 다른데서 쓰는게 불편하니 double [NFreq][CH * NT * 2] 이런 꼴로 하자

몇개를 바꿔야하나..

```cpp
    double **X_phi_temp, **kk_up, **kk, **X_delay;
    double **output;
    double **phi, **g, **X_temp, **AA;
    double **buf;
```

요 정도?    

데이터 구조를 좀 손봤는데 퍼포먼스는 반감. 오차는 많이 줄었지만 아직도 유효하게 크다. 실수부 허수부 오차가 같다. blas가 다 0 뜨네,
x_delay를 conjugate해서 연산한다. 

```CblasConjNoTrans``` 가 있었다. 테스트 성공했고 4 ~ 5배 빠르다. 

---

오디오 재생 마무리하자. 

pull 하고 conflict 해결하면서 뭔가 날라간거 같다. 

고치고 Rt_Output에도 이상한거 고치고 적당히 손봄. 동작 잘함.  뭔가 버그가 있을 거 같은데, 더 테스트 해봐야함. 

---

### 06.08

https://github.com/kooBH/IIP_Demo/issues/211

상단에 시계열 bar를 표시하고 싶은데 - KAnalysis 위젯의 가로 크기에 맞게 동적으로 길이 조절 되는 - , 쓸만한 위젯 베이스를 알아보자.  

indicator bar, time bar, 적당한 키워드부터 찾기가 어렵네,   
bar 겁색하니까 계속  Qt Data Visualization 만 나온다. 그냥 기다란 라벨을 써야하나. 기다란 이미지를 만들고 간격대로 draw 하고 이미지에 글씨쓸수 있을 테니까 시계열 표시하는 그런 걸로 가야하나? 일단 기본 위젯으로는 없는거 같다. 지금까지 검색한 거로는,

spectrogram들의 최대 길이를 잡아서 그거에 맞게 이미지를 - spec 생성 or 삭제시 갱신 - 만들어서 상단에 얇게 그려주면 되겠다. 
1. 상단의 버튼 레이아웃 상위에 버티컬 레이아웃 추가
2. 버티컬 레이아웃에 라벨 하나만 추가
3. 라벨 사이즈를 와 scale 에 대응하는 시간축 bar 이미지 만들기

### 06.09

https://github.com/kooBH/IIP_Demo/issues/212

MEMS 보드의 각각의 채널 쌍들이 뒤집혀서 들어옴. 
예를들어  
1 2 3 4 5 6 7 8 채널로 들어오면  
2 1 4 3 6 5 8 7 로 들어온다.  
```config.json``` 에 ```INVERTED_PAIR``` 옵션을 추가해서  
1 2 3 4 5 6 7 8 로 들어오게 뒤집는 기능을 넣으려함.  

```RT_Input``` 에서 입력버퍼를 각 채널에 분배할 때 뒤집으면 될 듯, 분기가 그럼 하나 더 생기긴 하는데.. 
퍼포먼스에는 별 차이 없으니까 일단 두자. 아니면 나중에 모듈 자체를 베이스기능만 있는 모듈하고 옵션들 주렁주렁 달린 모듈하고  
분할해도 되고. 

---

https://github.com/kooBH/IIP_Demo/issues/214

윈도우에서 재생하려고 하니, 장치 목록에 miniDSP가 떠있고 거기서 에러를 뱉으며 다른 장치들이 하나도 안 잡힌다. 

```
 *** Device List *** 

Current API : Windows ASIO

[0]miniDSP ASIO Driver
Probe Status = Unsuccessful
```

이렇게만 뜸

### 06.10

윈도우 드라이버 문제일 수도 있으니 realtek 드라이버를 최신으로 설치해보자. 

안됨.

---

rtauido를 cmake 돌려봄
```
-- Compiling with support for: wasapi
```

wasapi를 지원해야겠다. 
 
cmake 수정해서 돌아는 가는데 
+ 소리 엄청 깨짐. 
+ 샘플레이트도 기기마다 다를테니 - 노트북에 44.1K 지원안함 - 그걸 수동으로 설정해주거나 자동으로 하게 해야함. 

VS에서 디버깅을 하려고하는데 기타등등의 문제가 발생하는 거 같음. VS.. 호환성.. 제발..  

```RT_Output``` 재생만 시키고 뜯어 고쳐야겠다. 할당해제 없이 무의미한 할당을 계속하고 초기화를 중복해서 하는연산이 많다.

https://github.com/erikd/libsamplerate/blob/master/doc/win32.html
일단 보자

```
For Win32 there is a Microsoft Visual C++ compatible makefile in the Win32\ directory and a MSDOS batch file in the top level directory of the distribution.

To build the examples programs you will need to download the precompiled win32 or win64 libsndfile binary and install them.

Making the libsamplerate DLL on Win32 involves the following:

Using WinZip in the GUI, open the libsamplerate-0.X.Y.tar.gz file and extract the files into a directory. The following example assumes C:\.
In the directory containing the extracted files, find the file Win32\Makefile.msvc and open it in a text editor (ie Notepad or similar).
Find the line which starts with MSVCDir and modify the directory path to point to the location of MSVC++ on your machine. This allows the makefile to inform the compiler of the location of the standard header files.
Copy libsndfile-1.dll, libsndfile-1.lib and libsndfile-1.def from the directory libsndfile was installed in to the the directory containing libsamplerate.
Copy the header file include/sndfile.h from where libsndfile was installed to the Win32 directory under the libsamplerate directory.
Open a Command Shell and cd into the libsamplerate-0.X.Y directory.
Make sure that the program nmake (which is part of the MSCV++ package) is in a directory which is part of your PATH variable.
Type in the command
    C:\libsamplerate-0.X.Y> make
		
and press <return>. You should now see a a large number of compile commands as libsamplerate.dll is built.
To check that the built DLL has been compiled correctly type in and run the command
    C:\libsamplerate-0.X.Y> make check
		
which will compile a set of test programs and run them. If any of the programs fail the error message will be help in debugging the problem. (Note that some of the tests require libsndfile or libfftw/librfftw and are not able to run on Win32).
At the end of the above procedure, you will find the DLL, libsamplerate.dll, a LIB file libsamplerate.lib in the current directory. These two files, along with the header file samplerate.h (in the src\ directory) are all that you need to copy to your project in order to use libsamplerate.
```

할게 많았다.


---

윈도우에서 drag & drop 도 안되네, 첩첩산중.. 

```KAnalysis```에서 window 일때 open dialog에서 '/C:/~ ' 이렇게 / 가 앞에 붙음. 이걸 제거해야함. 일단 win32 분기로 해결함. 추후 Qt 버전이 다르면 고쳐질 가능성이 있음.  

### 06.11 

녹음시 노이즈가 엄청 심하다. 어디서 잘못들어가고 있는건가. 


+ UI에서 real time 스위치가 적용이 안되고 있음.  
+ Invert Pair 도 잘 안되는듯. 

real time 스위치 고침. 

녹음 문제 부터 해결 하자.

일단 MEMS 2 에선 2배 빠르게 녹음된다. 이건 invert_pair의 scope가 ambiguous 해서 생긴 문제, 해결함.  

노이즈는 하드웨어 문제로 판단

invert pair 관련해서는 일단 보류

---------------

확실히 그리는 UI를 개선시켜야겠다. 윈도우에서 버벅이는게 두드러진다. 

---

윈도우에서 재생시 힙이 손상되었다는 로그가 뜸. 메모리 문제가 확실히 존재한다. 

일단 샘플링을 주석처리하고 돌려보자.

play자체에 문제가 있는거 같기도? 

### 06.12

일단 VS 디버그 모드에서 KAnalysis 에서 에러 발생. 찬찬히 하나씩 해결해가자.

```cpp
  /* Scale ComboBox */
  combo_scale.addItem("1"),combo_scale.addItem("2");
  combo_scale.addItem("4"),combo_scale.addItem("8");
```

?? 왜 여기서 에러

striing 조작에서 에러가 나는데, ```xstring_insert.h```에서 단순히 char sequence 를 basic_string copy하는 코드에서 에러가 난다. 여기서 
```cpp
if (_State == ios_base::goodbit
			&& _Ostr.rdbuf()->sputn(_Data, (streamsize)_Size)
```

```_Data```를 못 읽는다. ```문자의 문자열을 읽는 동안 에러가 발생했습니다.```

디버깅 용으로 넣어둔
```cpp
 QObject::connect(this, &KComboBox::currentTextChanged,
                     [&](const QString &text) {
                       cout << text.toStdString() << endl;
                     }
   );
```

가 원인

이제 
```cpp
 for (json::iterator it = mod.begin(); it != mod.end(); ++it) {
      // std::cout << it.key() << " : " << it.value() << "\n";
      //KLabel *t1 = new KLabel(QString::fromStdString(it.key()));
여기--> KClickableLabel *t1 = new KClickableLabel(QString::fromStdString(it.key()));
      t1->setFont(font);
```

에서 에러

```처리되지 않은 예외 발생(0x00007FFF80ADA388, iip_demo.exe): Microsoft C++ 예외: std::bad_alloc, 메모리 위치 0x0000009006FCEC50.```

```
QString key_name = QString::fromStdString(it.key());
```

여기서 발생. 

```
std::string key_std = it.key();
여기서-->  QString key_name = QString::fromStdString(key_std);
```

https://stackoverflow.com/questions/30697200/qstring-throws-bad-alloc-exception

Qt디버그용 라이브러리를 따로 써야함.

---

재생시에 끝부분의 예외처리를 안해둔게 문제일까?
예외처리 해봄

이제 ```Qt5Core.dll``` 에서 에러가 발생. 지금 보니까 재생에 딜레이가 있는데 그거랑 indicator랑 
싱크가가 안 맞는거 같으니 일단 indicator를 끄고 테스트하자. 

그래도 터짐.

```oData->frameCounter = oData->size * sizeof(short);``` 엑세스 위반.

다음은

```oData->frameCounter += (frames * oData->channels  * sizeof(short));```

예외처리하고 적당히 돌아가는거 확인함. 테스트 해보자.
윈도우는 적당정당히 돌아감

---

리눅스에서 현재 RT_Output 인스턴스를 해제하는게 없어서 장치 해제가 안되서 에러.  해결하자. 

spectrogram 그리는걸 새로 짜고 재생하고 동기화시켜야겠다. 그거랑 같이 가야함.

---

재생이 현재는 전체 버퍼를 로드해서 돌리는데. 실시간으로 쌓는걸 추가해야할까?  
보류하자. 

--- 

윈도우에서 전체적으로 돌아는 가는데 제대로 돌아가질 않음. 재생도 이상하고 녹음도 이상하고. 

---

리눅스에서 48k 녹음은 잘되는데 재생하면 잘 안된다 것도 이상하고 


### 06.13  

위젯에 샘플레이트에 대한 고려가 하나도 없다. 음. 서로 다른 샘플레이트가 있는 파일이 같이 선택되는 경우를 어떻하지?  

이런 것들에 대한 고려를 해야겠다. 

샘플레이트 각 spec 마다 표시하고 전체 설정에 샘플레이트 추가하자. 

audacity 는 16k 48k 동시에 재생 가능, 이어폰이 16k 재생이 안되니까 샘플링을 한거 겠지. 

어디까지 해줄 것인가의 이슈가 생기네, 일단 단일 sample rate 만 지원 하자. 

출력 샘플레이트 어떻게 할지 생각하자.

---

윈도우 녹음이 이상함. 윈도에서 단일 채널 재생도 안되는거 같다. 

---

메모리도 쓸데 없이 많이 먹는듯, 500MB 라니. 무언가 해제가 잘 안되는거 같네, 


### 06.14

녹음시

```cpp
 if (status)
	  std::cout << "Stream overflow detected!" << std::endl;
```
구문 수행됨. 스트림 자체에 문제가 있는듯하다. 

rtaudio 기본 예제코드 돌려봄, 잘 녹음됨.

빌드시에 설정에 문제가 있는게 아닐까. 

기타 프로세스없이 녹음만 해도 같은 현상.  녹음 자체가 문제가 있다. 

빌드시 뭘 줘야하는가. 일단 원래 있던 것들 거의 다 끌어옴. 

안된다. 

녹음자체에 문제가 있음. 웨이브전환 안하고 raw 파일로 봤는데 그대로임.

-> 녹음이 문제다. 

예제코드를 프로젝트 환경에서 돌려보자. 같은 현상. 코드가 아니라 **빌드** 에 문제가 있다. 

이걸 어떻게 찾는가..?

간간히 정상적인 구간과 아닌 구간이 나타나서 살펴보니,  버퍼에 자리가 없는데도 가져오는거 같다. 

```cpp
/*
	* 매번 같은 크기의 버퍼를 읽지 않음. 보장할 수 없다. 
    * */
  iData->bufferBytes = frames * iData->channels * sizeof(signed short);
```

추가함. 테스트 해보자 

갑자기 시작하자마자 터져버리네 -> 샘플레이트 설정 문제

해결 된듯하다. 푸시 하자

### 06.17

표시 위젯을 opengl 쓰는 걸로 바꾸고, 재생과 표시를 동기화 처리해주자.


기존 상태에서 ```QXcbIntegration: Cannot create platform OpenGL context, neither GLX nor EGL are enabled``` 에러 발생

뭘 더 필요로 하는가. 

opengl 요구사항

```CMake
	    ${PROJECT_SOURCE_DIR}/lib/xcbglintegrations/libqxcb-egl-integration.so
	    ${PROJECT_SOURCE_DIR}/lib/xcbglintegrations/libqxcb-glx-integration.so
```
둘중 하나를 요구하는 듯. 일단 둘 다 포함함.

이미 프로젝트에 들어있었다. 

---

```QOpenGLWidget Class``` 를 정제해야겠다. base 클래스로 이리저리 사용할 수 있게 만들어보자.  


https://github.com/kooBH/Qt5_practice/blob/master/KGl.h 로 일단 하나로 통일해서 돌아가는 것 확인함   
https://github.com/kooBH/Qt5_practice/blob/master/main_graphic.cpp

---

KSpectrogram 위젯이 ```QOpenGLWidget Class```을 상속받게만 했는데 잘 돌아가는 것 같다. 대신에 메모리 관련 에러가 왕창생김. shared data 에러도 있는 걸 보아서 쓰레딩 관련되서 생기는 문제인듯. 

```Cannot make QopenGLContext current in a different thread``` 에러..

### 06.18

signal - slot 으로 하면 QT에서 해당 부분을 자체적으로 관리하니까 괜찮지 않을까?  

동적으로 connect 하고 disconnect 해야하네

https://doc.qt.io/archives/qt-4.8/qobject.html#disconnect

signal slot 호출도 잘 되고 값도 잘 넘기는데, 아무것도 그려지지 않는다.. 

label 이 아직 있어서인듯. 그려지질 않는다... 테스트 코드랑 양식은 같은거 같은데, 뭘 빼먹었는가. 

```Warning: When the paintdevice is a widget, QPainter can only be used inside a paintEvent() function or in a function called by paintEvent().``` 

PaintEvent에서만 호출해야하네, 그럼 buf 를 외부에서 처리하고 페인트 이벤트에선 버퍼를 그려줘야겠다.

그려는 지는데, 높이 조절이 안된다.
```  layout.setContentsMargins(0,0,0,0); ``` 가 잘 안먹히는가. 전체적인 그리는 방식도 다 ```buf```랑 ```buf_alt``` 쓰게 바꿔야한다. 

### 06.22

OpenGL 변환 완료하자. 

일단은 spec 크기가 설정이 안되는 문제부터. 

Aan 위젯단에서 간격조절이 안되는거 같다. 일단 border 다 그려서 보고 있는데 , SpecWidget 은 작은데 그 간격이 변하질 않는다. 

layout_spec의 문제인가 해서 setalign 했ㄴ는데 변화가 없다.

아니면 spec 자체가 크게 잡힌건가, algin top 하니 차이가 확실함. specWidget의 layout에서 margin 관리가 왜  
안되는 것일까. 

임시로 Label을 SpecWidget마다 넣었는데, layout_spec의 문제처럼 모양이 나옴. 
둘 다인가? layout_spec의 문제이면 SpecWidget에 align을 했을 경우 차이가 없어야하는데, 차이가 있음. 좀 더 변화를 줘 봐야겠다. 

해결

layout_spec을 top align 하고 margin 0, specwidget을 margin 0  

---

마우스 입력과 openGL 출력을 맞춰주자

## 06.24

입출력 다 맞췄는데 width가 고정되는 현상이 갑자기 발생. width 관련해서 몇개를 고쳤는 데, 그 중 하나이겠지. 

buf나 img나 width는 잘들어간다.   

다 되었다.   

---

불필요한 코드들을 제거하고, 전체적인 구성을 다듬자. 

spec에 img가 없어도 동작 가능하지 않을까.  
QPainter에 픽셀단위로 하는건 없는데 그냥 fill을 픽셀단위로 호출하면 되기는 

```cpp
for (int i = 0; i < ::loop; ++i) {
            QPainter painter(this);
            for (int x = 0; x < width(); ++x) {
                for (int y = 0; y < height(); ++y) {
                    const QColor color(static_cast<QRgb>(i+x+y));
                    painter.setPen(color);
                    painter.drawPoint(x, y);
                }
            }
        }
```

매번 Color를 만들어줘야함. 일단 놔두자. 어차피 그리고 바로 버리니까. 


---  

재생시 양 끝이 살짝 안맞는다. MousePress시 왼쪽으로 치우쳐짐. 
고침. 라벨에 맞게 되어있던 offset을 제거함. 

---

길이가 상이한 wav 를 불렀을 때, 레이어가 깨진다. 여러개 해보니 AlignCenter 모양이긴 한데..
```Qt::AlignLeft``` 해줘서 해결


---

```KAnalysis```에 추가적인 걸 달아주려한다 - 시간축 

길다란 이미지를 달아버릴까. 
+ spec 중에 width가 가장 큰거보다 길게해서 width 설정
+ shift size에 맞게 픽셀별 시간 계산해서 구간 별로 선도 긋고  ```drawText``` 하고
+ 축을 클릭하면 전체 채널을 선택할 수 있도록. 

좀 걸릴거 같은데 딜레이 할까. 

---

Input Device의 ComboBox ReProbe 수정.  

```combo.clear()``` 에서 stoi 관련 에러..

마지막 하나 남은 item을 제거할 때 에러 발생. 

```count()``` 가 1 일 때만을 예외로 둬서 돌려야 하는가. 

```
idx   cnt 
0  !< 0  add
1  !< 1  add
2  !< 2  add
      3
---------------
0  <  3  set
1  <  3  set
-2 <  3  rm   
      2
---------------
0  <  2 set
1  <  2 set
2  !< 2 add
      2
```

이렇게 해보자

해결됨.

---

한글 있는 거 못읽는 것을 해결하려 했는데, 리눅스에서는 잘만 읽힘. 윈도우 문제였었나.  

---

Windows 에서 verdigris가 안된다. 

### 06.27

3일정도만 verdigris  윈도우에서 해보다가 안되면 verdigris를 안쓰는 방향으로 가야겠다.  

https://code.woboq.org/woboq/verdigris/tutorial/tutorial.cpp.html  

W_OBJECT 쪽에서 나는게 아니라 signal, slot 쪽에서 에러표시가 남. 


https://stackoverflow.com/questions/40772958/macro-in-visual-studio-2015-expected-an-expression

역시 매크로관련해서 컴파일러 문제인가 


---

일단 msvc2019로 한번 돌려보자. 

MSVC2019 에서도 안됨.  

문제는 다른 윈도우에서 빌드되는 케이스가 있다는것.  뭐가 다른걸까.  


---

일단 없이 만들어서 빌드하고 하나씩 추가해보자. 

ㄱ냥 빌드가 안되네.. 첩첩산중


### 06.28

cross reference 구현 방식에는 문제가 없다 테스트 프로젝트 작동 잘됨.  

아무리봐도 또 visual studio 의 parsing 문제 같은데.. 컴파일러가 버그일꺼 같다. 

아니

1>c:\projects\cpp\iip_demo\include\k\KAnalysis.h(264): error C2086: 'int KSpectrogram::id': redefinition (compiling source file C:\Projects\cpp\IIP_Demo\include\K\KAnalysis.cpp)

```cpp
class KSpecWidget : public KWidget{
  //  W_OBJECT(KSpecWidget)
  private:
//    BorderLayout layout;
    QHBoxLayout layout;
    KWidget widget_west;
    QVBoxLayout layout_west;
    KPushButton btn_close;
    KLabel label_ch;
    KLabel label_sample_rate;
    KPushButton btn_export;
    KSpectrogram spec;
    KAnalysis*boss;

    int shift_size;
    int sample_rate;
    int length;
    /* wav buffer for each channel */
    short *buffer_wav;
    int id;

```

specwidget 클래스를 spectrogram 클래스로 보고 있다. 

vs 로 빌드할 수 있게, 하나로 되어있던 헤더를 3분할해서 떠먹여줌. 
빌드 성공.

---

SIGNAL, SLOT 에러는 그대로가 아니라 돌아가네. signal - slot 안쓴다고 함수 선언해둔거 충돌났었음

빌드 성공. 동작 확인.


### 06.29 

Mac 에서 시리얼 포트가

USB 3.1버스 - USB2.0 HUB - USB2.0 HUB  - CP2102 USB to UART Bridge Controller 로 뜨기는 하는데 시리얼포트라이브러리에서 인식을 못한다. 

https://developer.apple.com/documentation/iokit  


```cpp

    io_iterator_t serial_port_iterator;
    ...
    while ( (serial_port = IOIteratorNext(serial_port_iterator)) )
    ...

```


OSX 의 https://developer.apple.com/documentation/iokit iokit 라이브러리로 찾는데, 

USB 허브를 통해 연결되어서 iterate가 안들어 가는 걸까? 이쪽에 대해 좀 조사를 해봐야겠다. 


https://developer.apple.com/library/archive/documentation/DeviceDrivers/Conceptual/USBBook/USBDeviceInterfaces/USBDevInterfaces.html





IOKit 를 좀 살펴보자.  

일단 ```list_ports_osx.cc``` 에서 관련 있어 보이는 부분은

```cpp
    ...
    CFMutableDictionaryRef classes_to_match;
    io_iterator_t serial_port_iterator;
    ...
    mach_port_t master_port;
    ...

    kern_result = IOServiceGetMatchingServices(master_port, classes_to_match, &serial_port_iterator);
    ...
 
    while ( (serial_port = IOIteratorNext(serial_port_iterator)) )
     ...

```

인데 ```io_iterator_t``` 를 그냥 돌리는게 다라서 뭘 추가할 여지가 안보인다. 일단 기본상태엣 돌리면 
블루토스관련된거 하나만 검색되는 데 것도 연결이 안되어있어서인지 카운트만되고 출력은 안함.
== ```device.hardward_id``` 가 ```n/a```  

usb hub 를 io_iterator_t로 안잡는다는 것인데. 
일단 하드웨어 관리자에서 잡히기는 하니까. 그 경로만 찾으면 될것.  


```USBHub.h``` 가 존재한다. 

+ https://developer.apple.com/documentation/kernel/usbhub_h?language=objc
뭐 든게 없다.


+ https://stackoverflow.com/questions/16375078/how-can-i-keep-track-of-a-usb-device-across-disconnections
일반적으로 usb 포트가 연결되면 ```/dev``` 에 해당 포트가 생성된다고함. 하지만 생성되지 않음. 이것이 인식이 안되는 이유인가?  
usbhub로 연결된 것이 어떤걸로 인식이 되는가? serialport로 인식이 안된다면 현재 무엇으로 인식되고 있는지를 알아봐야한다. 

일단 유일하게 인식되는 ```Bluetooth-Incoming-Port``` 는 블루투스 항목에 잘 인식됨. 

추가로 현재 USB HUB 밑에 ```USB 2.0 BILLBOARD``` 라는 친구가 같이 연결되어있음. 

전체 구성을 보면  
기본값으로 

+ USB 3.1 버스 : AppleIntelCNLUSBXHCL
+ USB 3.1 버스 : AppleUSBXHCITR
+ USB 3.1 버스 : AppleUSBXHCITR

 이렇게 되어 있는데 

USB 허브 연결시 

+ USB 3.1 버스 : AppleIntelCNLUSBXHCL
    + USB2.0 HUB : VIA labs. Inc.
        + USB2.0 HUB : Genesys Logic. Inc.
            +  CP2102 USB to UART Bridge Controller - 사용하고자 하는 것
            +  USB 2.0 BILLBOARD
+ USB 3.1 버스 : AppleUSBXHCITR
    + USB3.0 HUB : : VIA labs. Inc.
+ USB 3.1 버스 : AppleUSBXHCITR

---

음성합성 작동 테스트 해야함. 

data 3개가 뭐였더라. 

```python
        saver1.restore(sess, "./save_new_ma/model.ckpt")
        saver2 = tf.train.Saver(var_list=var_list)
        saver2.restore(sess, "./save_new1/model.ckpt")
```

일단 요 2개 인가.

scp 로 그냥 다 받아오자
그전에 ssh 서버 설치

```
sudo apt-get install openssh-server
sudo service ssh start
``` 

일단 싹다 가져와서 돌리고 있다. 현재 출력물이 10.46초로 고정되어있는데, 이건 추후 조정하면되고 일단 합치고 보자. 

일단 CLI로 문장 입력하면 그거 재생하는 걸로 가보자. 그럼 동작방식이 어떻게 되야할까. 일단 파이썬 인터프리터는 껏다 키는게 안되니까 한번 열고 그걸로 계속 해야함. 

https://docs.python.org/3/c-api/init.html#sub-interpreter-support 우회적인 방법이 존재한다.  

+ 세인트 계정에 tts data 업로드함.

https://github.com/kooBH/IIP_Demo/wiki/Embedded-Python-in-CPP

일단 간단한 형태로 구현을 해보자

sample rate 는 왜 22050 이지.. 

파이썬이 켜져있으면 그냥 인터프리터 하나만 키도록 하자. main 에 한번씩만 호출하도록 함. 

아 GCP는 별도의 쓰레드에서 돌아가지.. Init 한데랑 API 호출하는 쓰레드가 다르면 안받아준다.. 

그려면 GCP를 Sub_Interpreter를 해주면 될까? 

이건 별도의 이슈로  https://github.com/kooBH/IIP_Demo/issues/243  

일단 tts 달긴 달음. 살짝 수정함. 함수형태로 해야 리턴을 기다리기 때문. 

1. 문자열을 입력받아서 
2. 파이썬의 인자로 보내고
3. 현재 시간으로 wav파일 생성하고 
4. 이름을 리턴받아서 
5. RT_Output으로 재생하는 걸로 가보자. 

일단 길이 조절이 뭔지 부터 -> 210으로 고정된 상수 변경하니 바뀜 .  

한글을 보내야하는데 음. 파이썬에서 디코딩해야하나 cpp에서 해야하나. 

갑자기 안돌아가네, 아까랑 코드는 같은데. 

일단 호출에서 호출이 안된다. 

중간에 쓰레드 관련해서 lock 안풀린게 있다. 주의 해야할듯

```unattended-upgrade=shutdown  --wait-for-signal```  

일단 이 친구들 kill 하고 돌려보자. 그대로임. 어디서 안풀리는 걸까. 그냥 재부팅 해야하나. 

다른 함수들 호출은 잘되는데 gen 만 그렇다. 인자의 문제인가. 

된다. 인자의 문제다. 인자를 전달하는 방식에서 문제가 있었는듯. 

C 단에서 유니코드 파이썬 오브젝트를 생성해서 인자로 넣어주자. 

해결함 웨이브 파일 생성 잘함. 근데 파일이 안열린다.  
이름도 맞는데, 경로가 어디서 바뀌는 건가. 
아니면 파일이 생성시에 안 닫히거나. 

작업디렉토리가 파이썬것 그대로 따라가네.  

일단 끝나면 cwd 복구시키게함.  

TTS에서 24비트를 생성하네.. 

기존에는 signed 16bit PCM만 사용해서 거기에다가 맞춰둠.  
문제는 TTS가 생성하는 파일은 완전히 다른 포맷이다. Floating Point 24bit PCM.. WAV랑 RT 모두 해당 포맷을 지원하게 수정하거나  
TTS 코드에서 바꿔줘야한다. 

---  

RT_Output 에 주의해야겠다. 샘플링 자체는 float 타입으로 변환해서 돌리는거라 입력자체가 float이면 그럴 필요가 없어진다. 

WAV 도 읽어보기 전까지는 float 인지 short 인지 모르는데 buffer를 어떻게 사용할까? 그냥 매번 switch 해야하는가? 


### 07.02

```WAV```의 버퍼를 ```void*```로 해두고 템플릿으로 ```reinterpret_cast```를 써서 사용하는 방식을 해보려함. 

이리저리 얽혀있는게 많아서 손볼것이 많다. 단순히 입출력에서 끝날 문제는 아니네 

전체 구성을 수정해야할지도.. short 타입의 입력만 고려해서 만들어왔으니.. 일단 대부분은 double로 되어있으니.  

short을 가지는 부분을 다 다양한 타입을 지원가능하게 해야겠다. 

+ WAV
+ RT
+ SpecWidget 의 버퍼 

말고는 나머지는 다 더블로 돌리니까 문제 없을 것. 

SpecWidget의 버퍼도 더블로 가지게 하면 문제 없을듯. 일단 나중에 하고.

---

일단 WAV 에서 파일을 읽을 때 타입을 파악해서 내부 루틴과 변수들을 해당 타입에 맞게 구성하는 건 너무 큰거 같으니 일단 현재해야할 것부터 해보자. 

WAV 단에서 안되네

http://www-mmsp.ece.mcgill.ca/Documents/AudioFormats/WAVE/WAVE.html

fmt_size 가 18이 들어온다. 이건 아예 헤더구조가 다르네.. 

(기록하지 않은 많은 작업)

WAV 를 non-pcm을 지원하게 변경함.  

OK. 리눅스에서 작동 확인함. 
현재 셈플레이트가 잘 안 맞는거 같다. 

수정함.
이제 윈도에서 해보자. 

---

윈도에서 pArg 가 null 이네.. 

PyUnicode_FromString 이 안됨 

파이썬 호출단에서 잘 동작하질 않는다.  

일단 한글인자를 보내는게 안되고 파이썬 호출도 잘 안된다. 

파이썬 코드 단독으로는 동작함. 

이슈는 한글을 어떻게 보내는가. 한글을 받는거는 GCP에서 이미 해보았기에 문제는 없다. 

### 07.03  

cpp -> python 한글 보내기가 잘 안되면 simpleCommand로 instruction string 을 직접 보내는 걸로 해야겠다.  일단 좀 더 해보고. 

경로는 2가지

+ ```PyObject_CallFunction``` 에 인자로 string 을 줘서 직접 한글을 보내는것
+ ```PyObject_CallFunction``` 에 인자로 PyObject를 줄 때, 한글 string을 Unicode PyObject로 만들어서 보내는것. 

문제는 한글을 넣은 오브젝트가 안만들어지고 한글을 담은 string이 보내지지 않는것.

```wstring```?
```cpp
#include <locale>
#include <codecvt>

   std::wstring hangul = L"한글";
   std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
   input = converter.to_bytes(hangul);

```

https://stackoverflow.com/questions/402283/stdwstring-vs-stdstring


>On Windows?
On Windows, this is a bit different. Win32 had to support a lot of application working with char and on different charsets/codepages produced in all the world, before the advent of Unicode.

>So their solution was an interesting one: If an application works with char, then the char strings are encoded/printed/shown on GUI labels using the local charset/codepage on the machine. For example, "olé" would be "olé" in a French-localized Windows, but would be something different on an cyrillic-localized Windows ("olй" if you use Windows-1251). Thus, "historical apps" will usually still work the same old way.

>For Unicode based applications, Windows uses wchar_t, which is 2-bytes wide, and is encoded in UTF-16, which is Unicode encoded on 2-bytes characters (or at the very least, the mostly compatible UCS-2, which is almost the same thing IIRC).

>Applications using char are said "multibyte" (because each glyph is composed of one or more chars), while applications using wchar_t are said "widechar" (because each glyph is composed of one or two wchar_t. See MultiByteToWideChar and WideCharToMultiByte Win32 conversion API for more info.

>Thus, if you work on Windows, you badly want to use wchar_t (unless you use a framework hiding that, like GTK+ or QT...). The fact is that behind the scenes, Windows works with wchar_t strings, so even historical applications will have their char strings converted in wchar_t when using API like SetWindowText() (low level API function to set the label on a Win32 GUI).

일단은  
```
       string -> bytearray    bytearray->unicoded        unicode -> string 
cpp ------------------------>       python         -------------------------->  cpp
```

로 해보려함. 

바이트어레이로 전송은 되는데 파이썬 단에서 unicode 로 전환이 안되네

c에서 한글 1글자를 2바이트 아스키로 생성, 이 포맷을 utf로 전환해야하는데.. 


```
bytearray(cp949 in Windows) -> utf-8 ->  [??? ]
```

c에러 return 받을 때 무슨 타입으로 받아야하는가.
----

긴 문장이 들어올 시에 완성될때까지 기다리지 않고 만들면서 재생을 할 수 있어야함. 

### 07.12

WPE blas 에 맞게 적당히 수정함, 테스트 예정

### 07.13

WPE_blas 코드 테스트

일단 blas는 간단하게 

https://github.com/kooBH/OpenAudioLibraryStudy/blob/master/manuals/CBLAS.md


```apt-get install libopenblas-base```
로 설치되는 거 사용. 

을 하려 했는데.

```/usr/lib/opebblas-base/``` 에 있는 라이브리르들이 링크가 안되네. 

```ldd``` 찍을 때 링크가 안되어있다. 

```/usr/include/cblas.h``` 도 인클루드가 안되네. 파일을 인식을 못하네.

cmake 단에서는 인식을 함.

---

몇가지 에러 수정함. 

### 07.14 

ubuntu package의 openblas 는 사용못함. ```CblasConjNoTrans```가 없다. git repo의 최근버전에는 존재함. 

https://github.com/xianyi/OpenBLAS  

받아서 default 로 make.  

open blas 헤더포함시킴. 원래 필요없는 걸로 알고 있는데, 연결방식이 잘못되었나. 일단 빌드는 되게 세팅해둠. 

자잘한 에러들 고치고. 작동테스트는 안해봄.

### 07.15

BLAS test 를 위해 check 수행. 

WAV::Append 에러 발생.   
Append 없이 돌리면 ```std::bad_array_new_length``` 발생 .

WAV 에서 fmt_type 가 Print 안됨. 

fmt_type 값이 쓰레기값이다. 잘못읽은 건지 처음에 잘못 작성한건지는 모르겠지만 fmt_type이 영향을 미치는 부분은 없다.  

잘못읽어서 전체적으로 깨지는건가? 이전에 추가한 부분은 fmt_type 아래 부분이라 거기 전까지는 변동사항이 없어야하는데.. 큰일 났구만.  

WAV 클래스를 좀 많이 손봐야겠다. 

일단 sample wav 를 받아서 그거 기준으로 동작시키자. 

읽을 때 값은 잘 들어간다. 

중간에 ```size_unit```이 0. 그러니 0을 써서 0을 리턴하니 Error.

새로 WAV 파일을 만들때 ```size_unit``` 값이 안들어가 있었다. 

아직 Print 할때 이상하게 출력됨.  그리고 현재 너무 느리다. 


sample wav 파일의 fmt_type이 이상하게 적혀있음 -> 고쳐야함

---

TestModule 

BLAS에서 예상했듯이 seg. 일단 valgrind 돌려보자. 

고칠게 많다. 수정하고 해보자. 

process 시작도 전에 죽어버리네. 

시작은 하게 함. 

4번 돌고 터짐. 

BLAS_1 해결.

valgrind 돌리면서 거진다 해결한거 같은데, ```buf ``` 가 고민이다.  
출력의 buffer 인데 현재 구조가 ```[Nfreq][Nch]``` 라 출력물로 옮길때의 for문이 좋은 형태가 아니지만.
이렇게 하지 않으면 중간 과정에서 형태가 더 나빠지고 그게 더 빈번하니까. 일단, 지금 형태로 가보자. 

일반 for 문들이 데이터 구조 변경하고 안 바꾼 것들이 많네. 

---

openblas 는 왜 ```/usr/lib```에 있는 so에 링크되어있나.. 

### 07.16

UI로 좀 보려 했는데 Convert 부분에서 seg. 

```raw[j][i] = raw[j][i + shift_size]; ``` 이 구문인데.  
여기 자체에서는 문제될 것이없다. 고친것도 없고. 

WAV 를 template 쓰지않고 switch 달음. 

frame_size 랑 shift_size가 설정이 안되어있었다. 

고침.

---

오차 1400..

속도차이도 없다.

연산이 틀려도 연산량 자체는 같은데..

행렬 곱자체가 비중이 너무 적어서 그런가. 대부부분 벡터연산이니까..
행렬 곱이라 해봤자 42x6 * 6 x 1 니까.. Fail

--- 

일단 ```std::bad_array_new_length``` 부터 해결하자. 
할당해제 문제 였음 해결함. 

---

https://github.com/kooBH/IIP_Demo/issues/247

TTS GUI

```  pValue = PyObject_CallFunction(pFunc, "s",input);```부분에서 인자가 안넘어가네.. 


### 07.18 

https://docs.python.org/3.5/c-api/index.html#c-api-index 

클래스 인스턴스에 관한 부분을 찾을 수가 없다. 명칭이 다른건가. 필요하지 않는 건가. 
애초에 인터프리터가 어떻게 돌아가는 건가

```int PyObject_IsInstance(PyObject *inst, PyObject *cls)``` 이게 있으니까 instance를 생성하는 것도 있을 텐데.  

https://stackoverflow.com/questions/9040669/how-can-i-implement-a-c-class-in-python-to-be-called-by-c

might be a lead? 

직접적으로 instance 를 받는 건아니고 명령을 수행하고 해당 명령의 리턴을 받을 수 있는 거 같은데, 이건 만드는 거라. 가져오는게 필요한테 좀 다른거 같다. 
```
  pClass = PyObject_GetAttrString(pModule,name_class);
  pInst= PyObject_CallFunctionObjArgs(pClass, NULL);

```

클래스 생성 성공. 

```
  pValue= PyObject_CallMethod(pInst,name_method,"s","hello world" );
```
method call 도 성공.

KTTS 에 달았고 인터페이스를 손보자. 

---

LineEdit에 한글이 안써지네. 

https://zapary.blogspot.com/2015/05/1504-fcitx-hangul.html

한글 입력기의 문제라네. setText로 한글을 출력하는 건되는데 입력이 안된다. 



https://yahwang.github.io/posts/36


```
libcomposeplatforminputcontextplugin.so
libcomposeplatforminputcontextplugin.so.debug
libibusplatforminputcontextplugin.so
libibusplatforminputcontextplugin.so.debug
libqtvirtualkeyboardplugin.so.debug

``` 
를 ```lib/platforminputcontexts``` 에 넣어둔뒤 링크해서 ibus에서 한글 입력해결. 

입력된 한글을 파이썬으로 받아서 보내는데서 인코딩 문제가 발생 중.


### 07.20 

Recorder - GUI  

+ RInput
+ KParam
달아둠

녹음관련 UI 틀만 달음.

녹음할 수 있게 코드 수정하고 테스트하면될듯. 

### 07.21 

녹음기능 달고, 동작 다듬자. 

UI 의 TTS를 TTS 옵션 활성시만 빌드. 

녹음기능 달고 빌드하고 윈도우에서도 몇가지 에러 수정함.

빌드해서 구글 드라이브에 올림. 문서 작업 중.


### 07.22

에러 발생 지점

```cpp
  stream_.deviceBuffer = ( char* ) calloc( deviceBuffSize, 1 );
```

샘플레이트가 어느 정도 이상이면 터진다. 

stream overflow 발생.
 
```
*******************************************************
** FAIL:This program is too slow! Fail to catch up!  **
*******************************************************
QObject::~QObject: Timers cannot be stopped from another thread

```

RECORD_TIME 을 10초로 해둠. 충분한 시간을 주자.

녹음 프로그램 만드는 건 어느정도 마무리가 되었다.


### 07.23  


TTS - GUI 

초기화 하는 걸 백그라운드 쓰레드에서 돌리자. 

KTTS에서 쓰레드를 돌려줄까

TTS 클래스 자체에 쓰레드를 쓰게해서 사용할까? 

TTS에 넣는것이 모듈화가 더 잘되었다고 할 수 있나? 

아니다 그냥 좀 걸릴겁니다. 메세지 띄워두면 될거 같은데. 그게 안들어가네. 

일단 디폴트 메세지로 해둠.

update는 signal 이 끝나야 적용. 

repaint를 써야겠다. 

---

인코딩 문제 유효함. 

고침. 윈도우 테스트는 해봐야함.

---

일단 TTS-GUI에서 생성과 재생의 동작을 확인하였다.  

인터페이스에 많은 수정이 필요하고, 동작 방식에 대해 좀 생각해봐야한다. WAV랑 RT_Output을 합쳐서 동작시키는 모듈을 만들어야 할 것 같기도 하고. 

일단 작동하는 동안 UI가 정지되는 것도, 문제는 없으나 동작하는 것을 알려줄 것이 필요할 것 같다. 

---

KInput 확장. RInput에 적용한 내용 적용함. 상속받도록 하고 싶은데 중간부분이 다른거라 형태가 이상할거 같음. 

KInput::RInput로 함. 

RInput에 채널별로 추가하자 

추가했으나 위젯이 많이 늘어서 번잡해졌다. 형태를 다듬자

### 07.24  

서버 우분투에 패키지 의존성이 깨져있다. 일단 내 pc에서 서버 테스트만 해야겠다.

flask 로 서버 돌릴 생각

https://github.com/kooBH/TTS-Server

### 07.25

일단 로컬 환경내에서 서버 - 클라이언트 동작 및 반응 속도를 확인해보자 

서버에 세팅 하는 중. 

jamo, jamotools, librosa, tensorflow==1.4.0, tqdm 설치함.  

scikit-learn 에서 설치가 잘 안되었다고 에러가 나네.

```it seems that scikit-learn has not been built correctly```

다 날리고 새로 venv.  

libcudnn 을 못찾겠다고 에러 발생.  

PATH랑 LD_LIBRARY_PATH 는 다 잘되어 있는데. 

설치한 cuda 8.0 에는 dnn은 포함되지 않음. 

cudnn설치했으나 so.6 를 찾는데 so.7이 설치되어서 그냥 일단 이름만 바꿔줌.

import 순서 바꾸니 scikit 문제 없어짐. 

tensorflow==1.4.0 설치함 (tensorflow-gpu==1.4.0 이 이미 설치되어 있는 상태)

PIL이 없다고해서

Pillow 설치함. (PIL 은 deprecated 되었다함)


---

서버 작업 중 다른거 하고 있자. 

PlayerWAV 모듈 을 만들자.

생성 즉시 재생하게 할까? 그게 깔끔할거 같은데, 지역변수로 만들게해서 동작하면. 

그러면 재생하는 동안 블락이 걸리게 된다. UI서 작동하는데 불편함이 생기게돔. 

따로 둬야겠다. 돌리고 재생중인지 체크하고.. 음

UI에서 재생을 누르면 별도의 쓰레드에서 재생을 하고 있겠지. 재생중에는 UI를 프로시저를 블락하지 않고. 있다가 재생이 끝나면 재생이 끝났음을 알려줘야겠지 아니면 UI에서 재생이 끝났는지 체크하고 있거나.  

일단 생성시 바로 재생하게 하고 block은 없게 하자. 체크하는 함수는 별도로 사용하게 해서 선택권을 주는 걸로 하자. 


### 07.26

libcur warpper 만드는 중. 
const char* 로 unicode를 다루자. 

utf-8 -> wcs 를 해야한다는 건가. 

http://jeremyko.blogspot.com/2018/01/libcurl-post-encoding.html

작동 확인.

인코딩 처리하는 거는 한글인 부분만 해줘야한다. 함수를 범용성 있게 짜려면 많은 부분을 손봐야할거 같은데.. 

일단 specific하게 하자. 

텍스트 -> TTS ->  WAV 동작 확인함.  

---

인터페이스를 구축하자.  서버 정보는 json에 담아두고 설정하는 인터페이스랑.. 아 로컬에서도 작동시킬 수도 있으니까. 둘 다 해줘야겠네..

KTTS를 거의 새로 짜야겠다. 

LineEdit에는 Hint로 설명해주자. 

```D_TTS``` 때와 아닐 때의 구분이 번거롭다. 깔끔하게 할 방법이 없을까?

### 07.29

일단 서버에서 받아와서 재생하는 것 부터 달아보자. 

일단 connect 부터 달아보자

connect 되고, 파일 받는 것도 됨.  

서버에서 받아서 재생은 되는 데, 두번째 전송부터 받기가 안된다. 

클라이언트 단에서 파일명이 깨져있었다. 수정함.  

메세지를 좀 더 추가하자.

추가함. 

---

로컬 TTS를 추가하자. 
추가함.

---

Spectrogram이 깨진다. fmt_type부분에서 이러는거 같다. 
output 작성에서 문제가 있는가? 좀 살펴봐야할듯. 

---

PlayerWAV도 추가해야한다.

---

https://doc.qt.io/archives/qq/qq27-responsive-guis.html

---

WAV에 유효한 파일인지 확인하는 함수 추가하자. 

--- 

너무 긴 문자열을 넘기지 않는다. 
```
하나하나하나하나하나하나하하나 하나하나하나하나하나하하나하나하나하나하나 하나하하나하나하나하나하나하나하둘둘둘둘둘둘둘둘둘둘둘둘둘둘 둘둘둘둘둘둘둘둘둘둘둘둘둘둘둘둘둘둘둘둘둘둘 둘둘둘둘둘둘둘둘둘둘둘셋셋셋셋셋셋셋셋셋셋셋셋셋셋셋셋 셋셋셋셋셋셋셋셋셋셋셋셋셋셋셋셋셋셋 셋셋셋셋셋셋셋셋셋셋셋셋셋셋셋넷넷넷넷넷넷넷넷넷넷넷넷넷넷넷넷 넷넷넷넷넷넷넷넷넷넷넷넷넷넷넷넷넷넷넷 넷넷넷넷넷넷넷넷넷넷넷넷넷넷넷다섯다섯다섯다섯다섯다섯다섯다 섯다섯다섯다섯다섯다섯다섯다섯다섯다섯다 섯다섯다섯다섯다섯다섯다섯
```

예외처리도 좀 많이 달아줘야겠다. 

### 07.30


```
이상은 작품 내에서 문법을 무시하거나 수학 기호를 포함하는 등 기존의 문학적 체계를 무시한 새롭고 실험적인 시도를 하였다. 이는 한국어 문학에서 이전에 시도된 적이 거의 없던 것이며, 이로 인해 그의 작품들은 발표 직후부터 현대까지 문학계에 큰 충격을 주었다고 평가된다. 또한 그의 작품은 줄거리의 전개방식이 명확한 경우가 많지 않고 소설의 전개는 극단적으로 주인공의 내면에만 치중되어 있는 자폐적 특징을 가지고 있다.주인공 역시 사회로부터 소외되어 자신의 흥미나 형이상학적 의미에만 집착하는 성향을 드러내는 경우가 많은데, 이는 작가 이상 스스로에 대한 묘사라고도 분석된다.문법파괴와 의식의 흐름 기법을 사용한 특유의 서술방식은 주인공의 비문법적인, 즉 무의식적인 내면을 잘 드러내며,기존 문학에 대한 반감 또는 무시를 의미하는 동시에, 서술의 대상을 없애고 언어 자체에만 비중을 둔다.

```

GET 은 길이제한이 있어서 POST로 바꿈. 

동작은 확인 하였으나 클라이언트 단에서 재생시에 bad_array_new_length 발생. 파일은 제대로 받음. 

gdb 로 돌리면 잘 돌아간다.. 음. 

Win10 수정사항 적용을 안한거였네. 

일단 문자열 길이는 다 동적으로 하게 해야겠다. 

### TODO
+ dynamic array length
+ real time TTS
+ fix float type bug

CppUrl,에서 배열 길이 동적으로 바꿈

---

```이 정도 길이는 한번에 가능합니다. 하지만 더 길어지면 나눠서 처리됩니다. ```

모델로 생성하는 것은 동시에 처리되니 실시간에서 제외된다쳐도, 변환하는 작업이 재생시간 보다 - 서버에서는 일단 - 빠르니까. 이 부분은 기다릴 필요없이 가능할 거 같다. 다만 동작 방식을 생각해봐야겠지. 
n개의 파일을 합쳐서 재생하거나 n번 재생하거나. 선택을 해야한다. 

n번 재생을 해야 주고 받기가 되겠지. 스트리밍은 그 다음단계이다.

---

POST_kor 고치다 말았네, 마저 고치자. 경로를 하드코딩해서 해놨다.  
고침.

---

윈도 RtAuido.cpp 에서  
```c++
 // close thread handle
  if ( stream_.callbackInfo.thread && !CloseHandle( ( void* ) stream_.callbackInfo.thread ) ) {
    errorText_ = "RtApiWasapi::stopStream: Unable to close callback thread.";
    error( RtAudioError::THREAD_ERROR );
    return;
  }
```

에서 재생 끝나고 에러 발생. 

### 07.31  

윈도우에서 재생 끝나고 에러 발생하는 상황의 조건을 모르겠다. 일단 처리는 해두었는데 종종 돌려보면서 체크해보자.  

---

서버와 클라이언트 각각 내부적으로 비동기 프로세스를 하면서 네트워킹을 해야한다.  

파이썬 부분의 구현은 방법이 다양한거 같은 데 좀 알아봐야겠다. 

+ https://github.com/celery/celery  
+ https://docs.python.org/ko/3/library/asyncio.html

1 : 비동기여야함
2 : 값을 중간중간에 받아 올 수 있어야함 
3 : wait/join 이 없어야함. -> flask에서 wait를 할 수는 없다.  

async multiprocess 

### asyncio.Task to “fire and forget”
+ https://stackoverflow.com/questions/37278647/fire-and-forget-python-async-await

서버단에서의 비동기 프로세스는 구현하였지만, 클라이언트가 하나일 때만 제대로 동작한다.  

클라이언트 별 데이터를 구성해야한다. 

이건 나중에 하자. 일단은 기능 구현 부터.   

https://github.com/kooBH/prac/tree/master/async_server_client

기능 동작 확인함. 많은 예외가 있겠지만 동작은 확인 하였다. 

+ 프로젝트 병합
+ 예외처리

어느걸 먼저 할까. 

프로젝트에서 동작할 때는 리스트에는 파일명만 넣어서 관리하고 서버에서 클라이언트로 주는 대로 클라이언트는 다운받아두고 process 하는게 좋을 것 같다. 

---

일단 여러 클라이언트가 동시에하는 경우는 없는 걸로하자. 아이피로 구분하거나 어케어케 구분하면 되는데 별로 필요가 없어 보인다. 

--- 


프로젝트에 합치자.  




### 8.1 

일단 프로젝트에 프로세스는 하지 않고 네트워크만 하는걸로 테스트 해보자. 

ip를 가지고 세션관리를 할 수 있지 않을까?

지금 asyncio 구현은 . 엄청 오래 걸린다. 1초 걸릴것이 42초 걸린다. 

결국 multiprocessing 을 해야하나? 그런데 프로세스가 다르기 때문에 데이터 공유하는게 잘 안될텐데.. 

파이썬에서 sequential 하게 보내게하자. 

---

이제 파일 전송을 달자.  

파일 전송 확인함 

----

클라이언트 단에서 재생을 할 때 쓰레드를 3개 돌리기 때문에 예외처리가 정말 많아야 할것이다. 

일단
1. 첫번째 웨이브 파일을 받으면 재생쓰레드 시작
2. 재생하면서 다른 웨이브 파일들을 메인 쓰레드에서 계속 받는다. 
3. 받아온 웨이브를 재생 쓰레드의 재생 목록에 append. 

+ 예외처리 할 요소가 많다. 

일단은 웨이브 파일일들을 버퍼에 쌓지않고 순차적으로 재생시켜보고 이상한지 보다. 

### 08.05

어디까지 짰더라. 

재생 단에서 버퍼 로드랑 오픈 스트림이 같이 있다. RT의 작동방식에 대해 좀 더 고려해봐야겠다.  

종료하는 부분만 어떻게 할 지 처리하면 일단 완료.  

```ThreadPlayerWAV```가 백그라운드에서 계속 작동하기 때문에 종료하는 시점이나 조건같은 걸 잘 고려해야한다.  

---

스트리밍을 위한 구상을 해보자. 
+ 버퍼를 두고 거기서 계속 소모

http://www.music.mcgill.ca/~gary/rtaudio/duplex.html

스트림 입력단과 버퍼를 연결해주면 된다.  

### 08.06

https://flask.palletsprojects.com/en/1.1.x/patterns/streaming/  

보내는 스트리밍은 괜찮은데 받는 스트리밍은? 

---  

일단 버퍼를 쌓아서 재생 할 수 있는 재생 창치부터 하자. 

+ 버퍼 크기
+ 재생 단위 : 리샘플링 해야하므로 최소 2048
+ 예외처리 : 비동기적이므로 

재생 단에서 리샘플링하면 다 float 이 되니까 그걸로 통일 시키자. 

일단 링 버퍼로한다음에 충돌하면 파라매터 조절하라는 메세지를 띄우자. 

동작은

커다란 float 버퍼, 작은 float 버퍼. 

입력받은 데이터를 샘플링해서 작은 float 버퍼에 적고, 이 작은 float 버퍼를 큰 float 버퍼에 append. 

재생은 커다란 float 버퍼를 순회하면서 동작.  

재생하는 순환이랑 쌓는 순환은 별도로 동작. 

RT_Output 을 좀 많이 고침. 작동하는 지 테스트 하고. 추가한 부분 작동하는 지 테스트 해야한다. 

기본 재생은 작동확인.

링 버퍼에서의 인덱스 관리가 생각이 필요하다.  

재생 인덱스와 저장 인덱스의 대소 비교로는 불가. - 순환하므로 - 

일단은 선형 버퍼를 가정하고 할까? - 원래 구상안이기도 함. 

=> atomic stock으로. RT_Input에 한것 처럼. 

인덱스 문제는 해결되었고. 이제 버퍼 크기를 어찌할까. 자리가 없으면 대기를 시킬까? 문제는 지금 데이터를 넣는 방식을 정하지 않아서.  
지금 설정하기는 난감하다.  

재생단에서 버퍼받아서 해야겠다. 

일단은 서버에서 경로받아서 하는 방식 유지. 그렇게 갭이 느껴지지 않네. 

---

이상적인 목표는 0.5초. 하지만 이미지 선명화를 위해 윈도우 크기를 최소 30은 해야 음질이 좋음. 윈도우크기가 10일때 첫 프레임이 0.8초 나머지가 0.3초 정도 걸림. 초기화 하는 부분을 처리하는 방법과 윈도우 크기를 낮게하면서도 음질을 좋게하는 방법 또는 선택을 해야한다. 

---  

출력 부분 변경 사항 적용 안된 부분들 다 적용 시키자.  


---

### 08.07

float format의 spec을 그릴때 값이 다 -3에서 놀고 있음. 

float 일때 min max 가 0 ~ 70.253

float
-521 ~ 71

short
-200 ~ 270

float 에서 어느 정도에서 min 을 잘라야할 지 모르겠다. 제대로 된 비교할 것이.. ? 

-250에서 걸러주었다. audacity에서 spectrogram 찍은 거랑 같은 모양이 나온다. 

---

fmt가 1로 들어오네. 수정함. float 타입도 재생됨. 

---  

float 타입에서 작동 전체적으로 잘됨. 전처리는 안해봄. 

---

audio 폴더를 ```lib/RtAudio```로 다 옮기자, 경로에서 현재 잘 안잡히네. 

### 08.08

윈도우 호환 작업. 

---

내부적으로 템플릿 함수를 래핑해서 사용하면 오버로딩으로 접근하되 구현을 템플릿으로 할 수 있다. 

### 08.19  

전체적으로 정리하자. 

일단 TheadPlayerWAV는 예외처리 안해도 파읽 읽기 단에서 예외처리 당해서 터질 테니까 보류.  

TTS관리는 프로젝트가 합쳐진 다음에 관리.

--- 

Run에서 실시간 부분과 wav 파일 읽어서 하는 부분을 분리하자. 

나누려면 좀 골치아프겠다. 

KApplication 과 Run이 서로 참조하기 때문에 같은 헤더를 사용한다. 동작하는 동안 다른 메뉴를 잠궈두기 위함인데, 이 기능을 빼고 상호 참조를 없애서 독립을 시켜두는게 편할 것 같다. 

KApplication 과 KRun 분리함.

```KRun``` -> ```KRealTime``` 으로 refactoring. 

```KProcessWAV``` 만듦. GCP 결과는 출력파일 append하는 방식으로.  

위젯단에서는 별로 하는게 없네. 거의다 ```UI_Module```에서 처리하도록 짜놨네. 

기록이 중요한것 같다. 내가 짜놨어도 기억을 못하니... 

Module 자체가 json으로 realtime 유무를 구분하게 해 두었는데, CLI라면 이게 필요하고 GUI라면 필요가 없다. 
Module의 생성자에 인자로 이 부분에 대한 처리를 해야겠다. 

Module을 분리하거나 조작하기 보다는 Widget쪽에서 swtich에 따라 동적으로 설정할 수 있게하면 될거 같다. 


<img width="715" alt="파일_001" src="https://user-images.githubusercontent.com/39723411/63244658-f9023600-c298-11e9-8d02-175d8e78db5c.png">




--- 

```RInput``` 만 있으면 될듯하다. ```KInput```으로 올려주자. 윈래 ```KInput```의 입력 파일 부분만 옮겨서 ```ProcessWAV```에 주자. 옮겨줌. 


### 08.20


뭔가 꼬인다. Module이 

GCP <-> UI_Module 로 긴밀히 연결되어 있고 거기다가 Process에서도 2가지 분기가 있게되서. 진행 흐름을 파악하기가 어렵네. 4개월전이 마지막 commit 이네. 

Module 부분을 많이 손봐야한다. 

파이썬 부분의 이슈가 좀 있네, 
https://github.com/kooBH/IIP_Demo/issues/278  

---


KProcess 계속.

wav input 일 때 계속 runnig 하네. 

---

뭐 좀 확인 하려 했는데 play 부분에서 double free

재부팅하니까 잘 돌아간다. 뭘 빼먹은 걸까.
버퍼 포인터 nullptr로 초기화안했었다.
고침

---

KProcess의 output 부분을 달아보자. 

GCP의 동작을 어떻게 할 것인가?  
실시간의 경우에는 text field에 계속 띄울 건데, GCP안 쓸때도 있고 wav input이면 어떻게 될지도 모르겠고, 음. 

---

spectrogram 생성이 엄청 느려졌다. cpu 문제인가 코드 문제인가. 다른 pc에서도 같은 현상. 코드 문제. 

이전 버전에서 가장 많이 달라진 부분이 버퍼 형성 부분인데 해당 부분을 위주로 살펴보자. 그건 spectrogram 생성과 관련이 없는데. 

idle 상태에서도 쓰레드 하나가 열심히 동작중. 이게 병렬성을 해쳐서 그런거 같다. 어떤 쓰레드가 이렇게 미친듯이 도는지 찾아보자.  

TTS에서

```cpp
            //TEST
            player.Run();
```

TEST라고 주석 달린친구 때문이네, TTS 손을 빨리 봐야겠다. 

ThreadPlayerWAV 먼저 봐야겠다. 저번에 짜다가 말아서 코드가 미완성 상태. 

적당히 고침 테스트 해보자. 

---

TTS 에러 발생

```하나입니다. 둘이오. 셋이라함 넷이다. 다섯입니다. 여섯이오. 일곱이라함. 여덟이다. 아홉입니다. 열이오.```

느리면 기다리게 해야하네. 

flask에서 생성이 좀 뒤쳐지면 기다렸다가 보내주게 하자. 

### 08.21

tts의 local 구현과 test가 문제. 일단 local 부분은 신경쓰지말고 구현하자. 

일단 코드 병합을 기다리자. 


---

나중에 코드를 다 src로 해야겠다. include 는 없애고. 엥간해서는 다 헤더에 inline으로 해되, qt 부분은 verdigris 때문에 따로 둬야하니까 .h 랑 .cpp 로 해두고. 

---


flaks에서 생성이 덜 된경우 어떻게 해야하는가? 기다렸다가 client에게 보내줘야지. 

문제환경 재현이 힘들어서 좀 나중에 하자. 

---

json 관리 클래스를 많이 다듬자. 유현하게하고 인터페이스도 많이고 치고. 

https://github.com/miloyip/nativejson-benchmark 이런 게 있네. 

https://github.com/kooBH/IIP_Demo/issues/280

어느정도 가변인자를 받을 수 있게 구현이 되었다. 

용법과 인터페이스를 고려해보자. 

알고리즘의 인자들을 double이지만 - int 도 double로 받아서 캐스팅 -  bool이나 string 을 사용하는 것들도 같은 클래스에서 파생해서 사용할 수 있게 하고 싶은데. 

전부다 

```cpp
// convenience type checkers
j.is_null();
j.is_boolean();
j.is_number();
j.is_object();
j.is_array();
j.is_string();
```

로 돌려서 해버릴까?  

---

TTS 는 local 없는 걸로 하고 싶으면 로컬에서 서버 돌리는 걸로. 

### 8.22

json을 파라매터부분만 관리하자. 나머지는 예외가 너무 많아서 메뉴얼하게 하는게 좋을둣. 일단 현상태 유지. 나중에 알고리즘 다 파라매터 넣을 때, 하자. 


---

Voice Conversion 을 병합해야한다. 

문제는 병합의 정도인데. 파이썬 인터페스를 구축하고 별도의 파이썬 프로젝트로 일괄 관리하는가. 아니면 c++ 프로젝트에 같이 포함을 시키는게. 환경문제 때문에 c++ 프로젝트에서 하는건 방법을 좀 생각해봐야한다. 

-> 서버에서 하는걸로. 

이를 위해 python server를 서브 모듈로 넣자. 리드미에 관련 사항을 추가하자. 

서브모듈 달아줌

---

버퍼관련해서 예외처리를 다 해준거 같은데, 코드가 안되어있다? 일단 수정하자. 

고침. 

---

v1.0 을 위해 뭐가 남았는가? 

1. 실시간 웨이브 플롯
2. param 관리
3. ?? 

별거 없네

### 08.23

v1.0 을 위해 뭐가 남았는가? 

1. wav input 시 작동. 
2. 실시간 웨이브 플롯
3. param 관리

v1.0은 힘들거 같고. 예외처리나 미흡한 부분들 해야하는데 방학내로는 1.0안될거 같다. 

v0.1.0 으로 하자. 

test 목적의 releaes 0.1.0 릴리즈함. 

---

wav 입력시 입력의 spectrogram을 올리고. output도 올릴까? 

출력 파일 생성 로그를 띄울까?. 그냥 로그만 띄우고 spec은 별도로 보게하자. 너무 난잡해진다. 

대신에 wav input 시엔 log에서 수행시간이나 입력파일에 대한 디테일한 정보도 넣어주는 걸로하고. 

log 지우는 버튼도 하나 달아야겠다. 

log는 textbrowser로 하고.

wav input 시 종료표시가 되지 않아서 해당 부분 처리. 
하면서 thread를 detach시켰다. 

이제 log를 띄우고 input load 버튼이랑 출력창 초기화 버튼 달자. 

label line_edit 의 구성을

btn 과 label 로 변경.  

이제 log 띄우는 기능을 달자. 

wav  가 생성됨을 알려하는데 vad에 따라 동작이 바뀐다.  

vad 가 켜져있으면 AfterVAD에서   아니면 AfterProcess에서 하면 될것 같다. 


---

KProcess의 레이아웃이 어그러짐. 코드상에서 이렇게 나타날 이유는 안보이는데, 하. 

---

그리고 spectrogram의 play시 어떤건 float이고 어떤건 short 버퍼일때의 처리는 안되있는 걸로 기억하는데. 

---

현재 모듈이 재사용이 안됨. 쓰레드가 procedure를 돌기 때문에 파읽 읽는 부분이 반복되지 않음.  

---

TextBrower.append가 안됨 ```qt cannot queue arguments of type qtextcursor``` 에러 발생


### 08.26

+ TODO
1. 재사용가능 한 Module
2. KProcess 레이아웃 문제
3. TextBrowser 연동. 
4. 디렉토리 재구축 
5. 종료시 파이썬 소멸자 호출

---

TextBrowser 보다는 TextEdit을 쓰는게 낫을듯. 하이퍼링크 기능을 쓰지는 않는다. 이게 좀 더 사용도 유연할 테고.
append 관련 에러는 그대로. 

connect 관련해서 qcursor가 등록이 안되어있어서 queue에 넣을 수 없다고 한다. 아마 이건 sequential 하게 돌아가는게 아니라. signal - connect로 쓰레딩해서 돌아가는 거 같다. 생각보다 복잡하게 돌아가네. 

아아. 해당 부분의 함수를 UI 모듈에서 호출하는데 이게 별도쓰레드에서 돌아가서 그렇네. signal-slot으로 연결을 해야하는데, 
근데 UI_Module은 qt 클래스가 아니다. GCP 부분은 또 어떻게 작동하는거지? 것도 별도 쓰레드인데. 

gcp 도 잘 되던게 같은 문제로 터져버리네. 뭐가 바뀌어서 이렇게 된건가, 

아 UI_Module이 Q_Object를 상속받아서 signal slot을 달면 되네. 

그러면 UI Module에 signal 달고, KProcess 에 slot을 달자. 

verdigris - signal 구현에 문제가 있다. KAnalysis에 구현한 대로 했는데 컴파일 에러가 뜨네. 

string이 안되는 건가. 

string 이 안되는 거였네. char*은 된다. 이걸로 보내자. 

char*로 구현하다보면 컴파일러 별로 좀 다르게 볼거 같은데. 안전하게 래핑해서 써야하나. 

const char* 로 간다. 이제 Broker 함수는 필요 없을듯,signal - slot 을 안 써서 구현한 함수니까. 

slot 이 반응하지 않는다. connect는 해주었는데, 포맷에 문제가 있는 걸까 ? 

```c++
  QObject::connect(module, &UI_Module::SignalLog,
      this,&KProcess::slot_log);

```

```c++
void UI_Module::SendLog(const char* _log){
  emit(SignalLog(_log));
  printf("SendLog : %s\n",_log);
}

```

emit이 들어가기는 들어가는데. 아 module 이 동적으로 관리되는 객체였다. 동적으로 connect 해줘야함

해결함.

---

모듈을 재사용가능하게 하자. 입력만 refresh 하는 함수를 Module에 달고 호출하면 될거 같다. 

json 관리 단에서도 reload를 추가해야하네. 

config json 먼저 해야 되겠다. 

---

1. herm  고유벡터 - first principle 위주로 : 이건 lapack 위주로 보면될거 같다. cgmm용

2. gpu로 돌아가는 wpe : python code . 서버에서 예제 코드를 돌려보자. 연구실 코드도 받아서 돌려봐야함.

3. 음성 변환 병합 : 파이썬 코드를 아직 받지 못함. 

코드 3개 다 받으면 작업량 폭발.. 


### 08.24

+ TODO
1. 재사용가능 한 Module - json configuration 재구축 이 선행
4. 디렉토리 재구축 
2. 종료시 파이썬 소멸자 호출 - 파이썬 관리 객체 - 전체적으로 좀 봐야겠다. 
3. 전체적인 레이아웃 조정

---

모듈에 Reload 추가. 

Reload를 하는게 아니라. procedure를 다시 하면 되는 거 아닌가? 

전체적인 Module의 동작에 대한 comment를 KProcess와 Module에 명시해야겠다. 일단 다시 파악하고. 

그전에 json 이 reload가 되야겠네.

json 과 config 관련된 코드를 싹 다 바꿔야겠네. 

인터페이스를 어떻게 할 까. 

param 과 input은 양식이 달라서 쓰기 힘들거 같다. 

사용 양식을 통일을 하고 싶은데, 예외사항이 계속 생기네. 

config를 바꾸는 데도 오래걸리겠지만 이에 맞게 기존 코드들 수정하는 것도 만만치 않게 걸릴거 같네. 

설정 파일 경로는 어떻게 괸리 할까. 전역으로 define 해둘까? 

일단 전역 - define - 으로 관리하자. 

Module에서의 사용도 좀 생각을 해봐야겠다. 

ConfigParam이 작동을 안하네. string 이 안들어간다.

돌아가게는 함. 



---

모듈을 손보자. 

Pause 가 아닌 Stop을 하도록

고치니까 slot이 작동을 안하네. 

signal 의 argument로 아무것도 안들어가서였다. signal 은 잘 보내지는데 argument가 들어갈때도 있고 아닐때도 있네.  

아ㅓ 일단은 편법으로,..,

```cpp

/*
 * signal 에 인자를 포함해서 보낼 시 
 * slot에서 인자를 받지 않는 경우가 발생함. 
 * 일단은 편법으로 KProcess의 변수에 직접 접근한후
 * signal로 update 호출.
 * */
void UI_Module::SendLog(const char* _log){
  printf("SendLog : %s\n",_log);
//  emit(SignalLog( _log ));
  krun->temp_log = std::string(_log);
  emit(SignalLogAlt());
}

```

---

include 폴더 없애기로함.  에러처리해야함. 

### 08.28 

include -> src 하면서 발생한 에러 fix. 

---

이상태로 0.2.0 release

---

일단 plasma 먼저 써보자. 

http://icl.cs.utk.edu/projectsfiles/plasma/html/InstallationGuide.html

설치 가이드. installer는 어디에? 

blas랑 hwloc을 설치해두고 경로를 넣어줘야하네. 

openblas는 지원 안하는가. 

걍 blas 옵션에 openblas 넣어주면 될거 같다. 

---

설치는 eigen이 더 편해보이는데 
헤더만 incl 하면 되서 끝나는데. 너무 간단해서 걱정이다. 컴파일 옵션도 없다. 
쓰레딩이 없는거 같은데?  
게다가 Eigen의 class 로만 연산이 된다. 

일단 고유벡터를 구하는 것의 성능만 구해놓자. 

---

json param 부분 window에서 에러. linux에서도 대충 땜빵해놓은 부분인데. 그건 문제가 안되고 다른 부분에서 또 문제 발생.

```cpp
std::ifstream ifs(file_path);
      json j = json::parse(ifs);
      auto it = args.begin();
      //string name = *it;
      string name = "PARAM"; // linux에서 첫 param 만 못 읽어서 땜빵해둔 부분
      std::cout<<name<<"\n";
      it++; // 이걸로 순회가 되질 않는다. 
      while(it!=args.end()){
      std::cout<<*it<<"\n"; 
        data[*it] = j[name][*it]["VALUE"].get<int>();
        it++;
      }
      ifs.close();
```

arg를 

```args = {"PARAM","CHANNEL","FRAME_SIZE","SHIFT_SIZE","SAMPLE_RATE","REFERENCE"};``` 이렇게 주고있는데 이게 문제일까?  

해당 에러는 다른 pc에서도 동일하게 발생 


```cpp
args = {"PARAM","CHANNEL","FRAME_SIZE","SHIFT_SIZE","SAMPLE_RATE","REFERENCE"};

	  for (string x : args)
		  std::cout << "args : " << x << std::endl;
```

args 에 string 이 안들어간다. 

args를 부모 클래스에선 잘 들어감을 확인했는데 상속받은 클래스에서 값이 없다. 
args 크기는 잘 받는데. 

+ https://akrzemi1.wordpress.com/2016/07/07/the-cost-of-stdinitializer_list/

initialize_list 자체에 문제가 있는 거 같다. 
std::vector로 다 바꾸니까 해결됨. 

---

기타 windows 에러 수정함. 

release를 해보자

```
심각도	코드	설명	프로젝트	파일	줄	비표시 오류(Suppression) 상태
오류		Could not read NSIS registry value. This is usually caused by NSIS not being installed. Please install NSIS from http://nsis.sourceforge.net	PACKAGE	C:\Projects\cpp\IIP_Demo\bin\EXEC	1	

```

에러발생. 지금 nsis 사이트의 다운로드 항목이 터져서 
https://sourceforge.net/projects/nsis/files/latest/download 에서 받음. 
돌아가네  
나중에 cmake 코드를 손좀 봐야겠다. 라이센스 달고. 설명도 좀 달고. 옵션도 좀 달고. 

---

circleCI의 python 항목에 Qt5를 ON으로 함. 현재 동작환경의 전제가 그러하다. 동작환경에 대한 재정립이 필요하다. 

일단 UI_Module로 인한 문제는 넘기고 일단 merge하자

### 08.29

Eigen Library Performance 구해두고 cmake랑 동작환경 정리하자. 

---

일단 고유값만 구해보자. 

Matlab : https://kr.mathworks.com/help/matlab/ref/eig.html
Eigen : http://eigen.tuxfamily.org/dox/classEigen_1_1ComplexEigenSolver.html

일단 matlab으로 돌린걸 파일로 꺼내서 다른 라이브러리들 테스트 할때 비교군으로 쓰자. 

파일은 생성함. 이제 Eigen 라이브러리를 합치자. 

test 코드는 돌렸는데, col/row major가 다른거 같다. 아니네, 그냥 뒤집혀 있는데. ??

일단 bin 파일은 col major 로 저장되어있다. 

상관없네. 고유값의 정의는 만족하니까. 

성능 테스트 해보자.  

그렇게 큰 차이가 안나네 어차피 사이즈가 작아서 별 영향이 없들듯. 

---

PLASMA 를 해보기 전에, 하던거 마무리 짓자. 

python 문제부터 


---

### TODO
+ menu 정리
+ 녹음 기능 추가 - CLI,Recorder,GUI
+ 파이썬 정리
+ CLI는 실행 옵션으로 
+ installer 손보기
+ 아이콘 달기
+ 리눅스 인스톨러

menu 좀 정리함. 아직 갈길 멀다.  
컴파일 옵션들을 다 정리함 Qt는 기본값이고 cli가 옵션, TTS 관련 부분 다 날림. 

top.h를 이제 안쓰도록 함. 모든 모듈은 자기가 쓸 거 자기가 include. 
bottom.h -> IIP_Algorithm.h  

### TODO
+ 녹음 기능 추가 - CLI,Recorder,GUI
+ installer 손보기
+ 아이콘 달기
+ 리눅스 인스톨러
+ json 파라매터 관리

check 부분 좀 다듬음. 
json으로 알고리즘 파라매터 관리하게 하는 부분 추가하자. json 단은 구현되어있다. GUI랑 연결 하는 것에서 좀 수정을 해야하나? 생각을 좀 해봐야겠다.

KParam.h 부터 좀 바꿔야겠네. json 한 항목당 한줄추가 이런식으로 함수를 짜야겠다. 그리고 Module에서는 그냥 바로 JsonConfig로 파라매터 받아오는 걸로 위에 짜놓을까. 아니면 상속받아서 별도의 클래스들 둘까. 보기는 상속받아서 하는게 모듈부에서는 좋지만 음. 큰차이는 없으니까 편한걸로 해보자. 

### 08.30

### TODO
+ 녹음 기능 추가 - CLI,Recorder,GUI
+ installer 손보기
+ 아이콘 달기
+ 리눅스 인스톨러
+ json 파라매터 관리

---

json을 상속받아서 클래스를 만들어 두는 게, 나중에 한데서 보기도 좋고, 엥간해서는 Module의 코드는 간략하게 가자. 

파라매터의 네이밍 규칙이 일관적이지는 않는데 이거는 원래 알고리즘의 변수명을 그대로 가져온 것이니까. 일단 보류. 

알고리즘 코드도 다 바꿔야한다.

일단 모듈단에서 static cast로 double -> int .

---

GUI를 수정하자. 

일단 KInput에서 좌측엔 기본 파라매터들을 옵션에서 고르는 것이니 ParamComboBox로 두고. 
나머지 알고리즘의 파라매터들은 TextEdit으로 두자. 

움. 전체적인 구상을 한 뒤에 짜야겠다. 

일단 모듈화를 해서 각 알고리즘별로 사용 할 수 있게 구성해야한다. 

레이아웃을 어떻게 할까? 현재는 grid로 되어있는데, 각 알고리즘 당 한줄을 넣으려고 한다. grid 써도 무방할거 같다. 한 col 씩 쓰면되니까.   

동작은, 일단 알고리즘 명을 받아야겠지. 아 알고리즘 명만 받으면 되네. 아니다. 명시는 해두는게 좋을까? 그냥 읽어버리면 뭐가 없어도 유효한지 알 수 가 없을 거니까?  

ParamTextEdit 위젯을 만들어야 겠다.

input으로 file_path, algo_name, key를 받도록.  
이게 거슬리는게 모든 json 관련 위젯 하나하나 마다 json파일을 열고 닫는다. 비효율이다. 하지만 json파일을 수정할 떄 각 항목마다
json파일을 열고 닫도록 해야하기 때문에 이렇게 해두는 것도 괜찮은거 같다. 

알고리즘의 파라매터가 시술형과 정수형이 다 있는데, 현재는 다 double로 하고 int인 것은 알고리즘의 파라매터에서 정수로 받게해서 캐스팅을 하고 있다. 문제는 json에서 실수형으로 관리되기 때문에 파일 안에서 10이어도 10.0으로 표기될 것이고 GUI에서 입력 받을 때도 구분을 할 수가 없게 된다. 이 문제를 어떻게 다룰 것인가?  

json파일에 정수형 항목에 대한 item을 추가해서 구분하게 하면 되는 되지만 괘나 편법이다. 근데 괜찮아 보이기도 하고..? 
알고리즘에 ```__integer__``` 항목을 추가하였다.  

아 TextEdit아 아니라 LineEdit이어야하지 .바꾸자. 바꿈. 

LineEdit이 json을 write 하게 하자.  
최상단에 무엇에 관한 파라매터인지를 표기해야겠다.  

json 파일을 수정하게 해야하는데, 어떤 이벤트를 기준으로 수정해야할까? 엔터? 아니면 유효한 double / int 값이 들어왔을 떄? 

---

파라매터를 구분하도록 스타일을 적용해야겠다. 



---

현재 WPE가 아닌 알고리즘은 생성자에 param을 받지 않고있다. 


### 09.02 

### TODO
+ 녹음 기능 추가 - CLI,Recorder,GUI
+ installer 손보기
+ 아이콘 달기
+ 리눅스 인스톨러

Widget에서 integer 인지 아닌지 명시를 하자. 추가함. 

LineEdit에서 변경시 double integer 유효성 확인 후 수정사항 적용시키자. 


TextEdit 시 보다

```
void QLineEdit::textChanged(const QString &text)
This signal is emitted whenever the text changes. The text argument is the new text.

Unlike textEdited(), this signal is also emitted when the text is changed programmatically, for example, by calling setText().

Note: Notifier signal for property text.
```

추가함. 

---

PLASMA 를 써보자.
v19 를 써보려 했는데, omp 4.5 이상을 요구한다.  
v17은 cmake가 없네. 
v18 도 omp 4.5 이상을 요구하네, Window 에서 2.0 버전 그것도 불완전한 2.0 까지만 지원하는데.. 

blas를 필수적으로 까는 거 이전에 여기서 문제가 있을 수 있다. 

ICC를 쓰면 되는데,, ICC 쓰면 verdigris가 안되고.

```
make[2]: *** [CMakeFiles/iip_demo.dir/src/K/KSpectrogram.cpp.o] 오류 2
/home/kbh/git/IIP_Demo/lib/verdigris/wobjectimpl.h(755): error: expression must have a constant value
      constexpr static Arrays arrays = buildArrays();
```

```cpp
  constexpr static auto buildArrays() {
        auto r = Arrays{};
        DataBuilder b{r};
        generateDataPass<T>(b);
        return r;
    }
```

```constexpr``` 이 맞는데. 

선택지
1. 다른 egien 라이브러리 추가 조사
2. verdigris fix
3. eigen library 사용
4. ???

문제점
1. OpenMp > 2.0   < - > MSVC  ==> ICC로 해결
2. Verdigris(GUI) < - > ICC   ==> ???

---

openblas 써보자. 

lapacke.h 를 써야한다. 저번에 어떻게 썼더라? lapacke.h 는 어디서 가져온 거고?

```lapack-netlib/INCLUDE``` 안에 들어는 있는데 이게 맞는 건가? 일단 그거 가져와서 하는데 되는 거 같다. 

문제는 lapack 에 함수 종류가 많네 

```
zheev
zheev_2stage
zheevd
zheevd_2stage
zheevr
zheevr_2stage
zheevx
zheevx_2stage
zhegv
zhegv_2stage
zhegvd
zhegvx
```

다 고윳값을 구한다는 거 같은데, 조금씩 다르다. 


+  ZHEEV
: computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A.

+  ZHEEV_2STAGE
:  computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A using the 2stage technique for
 the reduction to tridiagonal.

+ ZHEEVD 
: computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A.  If eigenvectors are desired, it uses a
 divide and conquer algorithm.
 The divide and conquer algorithm makes very mild assumptions about
 floating point arithmetic. It will work on machines with a guard
 digit in add/subtract, or on those binary machines without guard
 digits which subtract like the Cray X-MP, Cray Y-MP, Cray C-90, or
 Cray-2. It could conceivably fail on hexadecimal or decimal machines
 without guard digits, but we know of none.

+ ZHEEVR
: computes selected eigenvalues and, optionally, eigenvectors
 of a complex Hermitian matrix A.  Eigenvalues and eigenvectors can
 be selected by specifying either a range of values or a range of
 indices for the desired eigenvalues.

 ZHEEVR first reduces the matrix A to tridiagonal form T with a call
 to ZHETRD.  Then, whenever possible, ZHEEVR calls ZSTEMR to compute
 eigenspectrum using Relatively Robust Representations.  ZSTEMR
 computes eigenvalues by the dqds algorithm, while orthogonal
 eigenvectors are computed from various "good" L D L^T representations
 (also known as Relatively Robust Representations). Gram-Schmidt
 orthogonalization is avoided as far as possible. More specifically,
 the various steps of the algorithm are as follows.

 For each unreduced block (submatrix) of T,
    (a) Compute T - sigma I  = L D L^T, so that L and D
        define all the wanted eigenvalues to high relative accuracy.
        This means that small relative changes in the entries of D and L
        cause only small relative changes in the eigenvalues and
        eigenvectors. The standard (unfactored) representation of the
        tridiagonal matrix T does not have this property in general.
    (b) Compute the eigenvalues to suitable accuracy.
        If the eigenvectors are desired, the algorithm attains full
        accuracy of the computed eigenvalues only right before
        the corresponding vectors have to be computed, see steps c) and d).
    (c) For each cluster of close eigenvalues, select a new
        shift close to the cluster, find a new factorization, and refine
        the shifted eigenvalues to suitable accuracy.
    (d) For each eigenvalue with a large enough relative separation compute
        the corresponding eigenvector by forming a rank revealing twisted
        factorization. Go back to (c) for any clusters that remain.

 The desired accuracy of the output can be specified by the input
 parameter ABSTOL.

 For more details, see DSTEMR's documentation and:
 - Inderjit S. Dhillon and Beresford N. Parlett: "Multiple representations
   to compute orthogonal eigenvectors of symmetric tridiagonal matrices,"
   Linear Algebra and its Applications, 387(1), pp. 1-28, August 2004.
 - Inderjit Dhillon and Beresford Parlett: "Orthogonal Eigenvectors and
   Relative Gaps," SIAM Journal on Matrix Analysis and Applications, Vol. 25,
   2004.  Also LAPACK Working Note 154.
 - Inderjit Dhillon: "A new O(n^2) algorithm for the symmetric
   tridiagonal eigenvalue/eigenvector problem",
   Computer Science Division Technical Report No. UCB/CSD-97-971,
   UC Berkeley, May 1997.


 Note 1 : ZHEEVR calls ZSTEMR when the full spectrum is requested
 on machines which conform to the ieee-754 floating point standard.
 ZHEEVR calls DSTEBZ and ZSTEIN on non-ieee machines and
 when partial spectrum requests are made.

 Normal execution of ZSTEMR may create NaNs and infinities and
 hence may abort due to a floating point exception in environments
 which do not handle NaNs and infinities in the ieee standard default
 manner.

+  ZHEEVX
 :  computes selected eigenvalues and, optionally, eigenvectors
 of a complex Hermitian matrix A.  Eigenvalues and eigenvectors can
 be selected by specifying either a range of values or a range of
 indices for the desired eigenvalues.

+  ZHEGV
:  computes all the eigenvalues, and optionally, the eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.
 Here A and B are assumed to be Hermitian and B is also
 positive definite.

+  ZHEGVD 
: computes all the eigenvalues, and optionally, the eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.  Here A and
 B are assumed to be Hermitian and B is also positive definite.
 If eigenvectors are desired, it uses a divide and conquer algorithm.

 The divide and conquer algorithm makes very mild assumptions about
 floating point arithmetic. It will work on machines with a guard
 digit in add/subtract, or on those binary machines without guard
 digits which subtract like the Cray X-MP, Cray Y-MP, Cray C-90, or
 Cray-2. It could conceivably fail on hexadecimal or decimal machines
 without guard digits, but we know of none.

+  ZHEGVX 
 : computes selected eigenvalues, and optionally, eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.  Here A and
 B are assumed to be Hermitian and B is also positive definite.
 Eigenvalues and eigenvectors can be selected by specifying either a
 range of values or a range of indices for the desired eigenvalues.

일단 가장 기본 형태부터 써보자. 


```fortran
subroutine zheev	
(
character 	JOBZ,
character 	UPLO,
integer 	N,
complex*16, dimension( lda, * ) 	A,
integer 	LDA,
double precision, dimension( * ) 	W,
complex*16, dimension( * ) 	WORK,
integer 	LWORK,
double precision, dimension( * ) 	RWORK,
integer 	INFO 
)	
```

```cpp
 LAPACK_zheev( &jobz, &uplo, &n, a, &lda, w, work, &lwork, rwork,
                      &info );
```

```

[in]	JOBZ	
          JOBZ is CHARACTER*1
          = 'N':  Compute eigenvalues only;
          = 'V':  Compute eigenvalues and eigenvectors.


[in]	UPLO	
          UPLO is CHARACTER*1
          = 'U':  Upper triangle of A is stored;
          = 'L':  Lower triangle of A is stored.
[in]	N	
          N is INTEGER
          The order of the matrix A.  N >= 0.
[in,out]	A	
          A is COMPLEX*16 array, dimension (LDA, N)
          On entry, the Hermitian matrix A.  If UPLO = 'U', the
          leading N-by-N upper triangular part of A contains the
          upper triangular part of the matrix A.  If UPLO = 'L',
          the leading N-by-N lower triangular part of A contains
          the lower triangular part of the matrix A.
          On exit, if JOBZ = 'V', then if INFO = 0, A contains the
          orthonormal eigenvectors of the matrix A.
          If JOBZ = 'N', then on exit the lower triangle (if UPLO='L')
          or the upper triangle (if UPLO='U') of A, including the
          diagonal, is destroyed.
[in]	LDA	
          LDA is INTEGER
          The leading dimension of the array A.  LDA >= max(1,N).
[out]	W	
          W is DOUBLE PRECISION array, dimension (N)
          If INFO = 0, the eigenvalues in ascending order.
[out]	WORK	
          WORK is COMPLEX*16 array, dimension (MAX(1,LWORK))
          On exit, if INFO = 0, WORK(1) returns the optimal LWORK.
[in]	LWORK	
          LWORK is INTEGER
          The length of the array WORK.  LWORK >= max(1,2*N-1).
          For optimal efficiency, LWORK >= (NB+1)*N,
          where NB is the blocksize for ZHETRD returned by ILAENV.

          If LWORK = -1, then a workspace query is assumed; the routine
          only calculates the optimal size of the WORK array, returns
          this value as the first entry of the WORK array, and no error
          message related to LWORK is issued by XERBLA.
[out]	RWORK	
          RWORK is DOUBLE PRECISION array, dimension (max(1, 3*N-2))
[out]	INFO	
          INFO is INTEGER
          = 0:  successful exit
          < 0:  if INFO = -i, the i-th argument had an illegal value
          > 0:  if INFO = i, the algorithm failed to converge; i
                off-diagonal elements of an intermediate tridiagonal
                form did not converge to zero.

```

BLAS 에서는 복소수도 double 2개로 받더니, 여긴 또 ```__complex__ double```을 달라고한다. 

일단 캐스팅해서 돌려보자. 돌아가기는 하고, 멀티 쓰레딩도 된다. 

데이터 읽어서 연산은 시켰으나. 검증하는 부분을 구현해야겠다. 

데이터 넣는 부분부터 틀렸다. 
행렬이 두 알고리즘 모두 잘 안들어간다. 

### 09.03

N = 4 가지고 테스트 하자

```matlab
   A
   1.2589 + 0.0000i   1.0763 - 0.5200i   0.1690 - 0.2269i   1.7411 - 0.6080i
   1.0763 + 0.5200i  -1.6098 + 0.0000i   0.4868 - 0.1828i   0.0645 - 1.5256i
   0.1690 + 0.2269i   0.4868 + 0.1828i  -1.3695 + 0.0000i   1.5417 + 0.6276i
   1.7411 + 0.6080i   0.0645 + 1.5256i   1.5417 - 0.6276i  -1.4325 + 0.0000i

   V
   0.2812 + 0.0093i  -0.1539 + 0.0352i  -0.3312 + 0.4423i  -0.6516 + 0.4074i
  -0.1814 - 0.4834i   0.6173 + 0.2624i  -0.2216 - 0.3752i  -0.2401 + 0.1899i
   0.3595 + 0.2247i   0.5886 - 0.3612i   0.5137 + 0.0752i  -0.2598 - 0.0792i
  -0.6889 + 0.0000i  -0.2196 + 0.0000i   0.4850 + 0.0000i  -0.4919 + 0.0000i

   D
   -4.1976         0         0         0
         0   -1.5736         0         0
         0         0   -0.2949         0
         0         0         0    2.9132

   sum(A*V  -  V*D,'all') = -1.8180e-15 + 4.7531e-16i

```

일단 eigen Library 의 A*V - V*D 가 0에 근사함.

---

OpenBLAS의 LAPACK을 해보자. BLAS 출력을 잘못하고 있었다. idx 초기화를 안했네. 

OpenBLAS의 LAPACK이 잘 되는지 검증을 해보자. 

cplx mat * cplx mat  - cplx mat * real vec  을 해야하는데  
행렬 곱은 blas 쓰고 행렬x벡터는 for문 그냥 돌리자.  

zheev 가 inplace 였다. 복사도 해야하네. 
아니다 inplace 시키는게 hermitian 이라 대칭이므로 Upper 또는 Lower에서 넣어준다는 것. 그러면 대칭행렬 곱 연산 루틴이 있다면 그걸 쓰면 되지 않을까 싶은데 생각해보니까 대각성분이 바뀌니까. 안될거 같네.

복사해서 해야겠다. 

cblas하고 lapacke 하고 같이 include 하니까 충돌나는데, 

OpenBLAS 빌드시에 LAPACKE 을 쓴다는 옵션을 줘야했던가 같은 실낱같은 기억이 스쳐지나갔다. ```iip_sph_pp``` 를 진행할 떄는 apt-get 으로 패키지 openblas를 사용했던것 같다 - lapack 포함 빌드인 - 

상당히 꼬일 거 같은데.. linux에서 이걸 해결해서 빌드하고 링크한다 쳐도 윈도우에서는 더 복잡하게 가야하지 않을까? 아니다 cmake에 MSVC 옵션이 있기는 하니까? 근데 얘들 윈도우에서 잘 안되잖아. 

조사를 더 해봐야겠다. 

https://github.com/xianyi/OpenBLAS/wiki/How-to-use-OpenBLAS-in-Microsoft-Visual-Studio  

된다는 거 같다. 

그럼 OpenBLAS가 lapack을 같이 쓸 수 있게하는 옵션을 찾아보자.

일단 OpenBLAS안의 lapack 폴더는 그냥 netlib 의 lapack을 냅다 들고온 거같다. 빌드도 안되네 이건 옵션을 제대로 안줘서 인거 같다.  

아니면 편하게 apt-get으로 한거를 파일 가져와서 링크하며 되지 않을까?  

https://github.com/gogyzzz/iip_sph_pp/issues/91  아.. 좀 더 자세히 읽었어야 했네. 여기서 한 대로 다시 해보자.  

make install 해서 이동된 헤더들만 사용. 빌드는 성공하였다. 이제 다시 테스트로 돌아가자.  

테스트 성공. 

---

Eigen 과 Lapack  모두 오차가 최대 e-14 이다 대체로 e-16 정도 되는 것 같다. 


lapack 이 쓰레딩도 하고 최적화도 되어있어서 10배 빠르다. matlab 과는 6이상일 떄는 많이 격차가 나지만 그 이하에서는 거의 차이나지 않는다. 

ZHEEV_2STAGE 이랑 ZHEEVD 만 해보면 될거 같다. 다른 것들은 같은 알고리즘에 좀 더 특정적인 인터페이스로 보임.  

일단 ZHEEVD 먼저 해보자.  

---

openblas를 submodule 로해서 환경에 맞게 빌드하도록 해야겠다. 설치파일로 쓸거는 어쩔 수 없을 거 같긴한데. 

OpenBLAS에 dynamic으로 하는게 있던게 그것도 한번 알아봐야겠다. 

### 09.04

#### TODO
+ ZHEEV_2STAGE
+ ZHEEVD
+ BLAZE
+ PLASMA - QUARK
+ lapack 래핑. 

---

zheev_2stage 는 OpenBLAS 에서 구현이 안되어있네,

LAPACKE_zheevd 는 zheevd 랑 2stage 둘 다 구현이 되어있다. 

zheevd 가 파라매터가 다르고 integer array work가 추가적으로 필요하다.  

첵크해서 추가할까. 아니면 그냥 사이즈 받을 때 할당 해버릴까. 아니면 상속 구조로 갈까? 

상속 구조로 가는게 나은거 같기도 하네. 

zheev -> zheevd 상속.  

---

zheevd 랑 zheev 가 그렇게 차이가 없다. size 2, 4 일 때 빠른 알고리즘이 서로 바뀌긴하나. 유의미한 차이를 내지 못한다. 

zheevd_2stage 가 ```symbol lookup error: undefined symbol : zheevd_2stage``` 가 뜬다. 구현이 안된걸까. 내가 링크를 잘 못한 걸까. 오타를 낸걸까.  

https://github.com/xianyi/OpenBLAS/blob/ce3651516f12079f3ca2418aa85b9ad571c3a391/lapack-netlib/LAPACKE/src/lapacke_zheevd.c

https://github.com/xianyi/OpenBLAS/blob/ce3651516f12079f3ca2418aa85b9ad571c3a391/lapack-netlib/LAPACKE/src/lapacke_zheevd_2stage.c

보니까 내가 쓰고 있는 함수랑 인자가 다른데..? 뭔가 다른걸 링크해서 쓰고 있는 건가. 뭘 가져다 쓰고 있는 거지.  

---

역행렬 구하는 것 처럼. 고윳값 구하는 것도 코드짜여진거-낮은 채널- 있으면 찾아보자.

일단 blaze 에는 안보이는 거 같다. 

이건 좀 구하기 힘들거 같은데, 

---

### TODO
+ BLAZE
+ PLASMA - QUARK(구버전)
+ (openBLAS를 쓴다면)openblas set-up script 
+ 일반적인 역행렬 함수

blaze 를 일단 써보자. 

blaze is mere wrapping of blas/lapack.  옛날에도 blas에서 사용하려고 해서 알아보니까. 그냥 함수 래핑이었다. 결국엔 어떤 blas를 사용하는가의 문제이지 이게 뭘가를 해줄 거 같지는 않다. 

그럼 일단 이정도 까지만 하자

코드 짠거를 프로젝트에 합치자. 그리고 원래 프로젝트 작업 계속하자. 

BLAS/LAPACK을 처리하는게 급선무겠다. 
https://github.com/kooBH/IIP_Demo/issues/302  


---
### 09.05

### TODO
+ BLAZE
+ PLASMA - QUARK(구버전)
+ (openBLAS를 쓴다면)openblas set-up script 
+ 일반적인 역행렬 함수


역행렬 구하는 루틴을 만들자. 

일단  blaze 에서 가져와서 변환할까 iip_sph_pp 에서 가져와서 변환할까 부터 정하자. 

둘 다 별 차이는 없을 듯. 각자의 타입을 사용하도록 되어있는데. 나는 그냥 double 2개를 re,im 으로 해서 raw하게 할 것인기 떄문에 
어차피 다 변환해야하는 건 같다. 커스텀한 타입을 사용하는 것은 그 타입을 계속 사용한다는 전제하에서는 굉장히 편리하지만, 그렇지 않을 경우 귀찮고 까다로워진다. 그냥 dobule 형으로 해도 직관적으로 받아들이기 쉽기 때문에 이렇게 간다. 

일단 복소수만 하면 되니까 복소수 행렬만 변환하는 스크립트를 짜보자. 전에 짠게 남아있으면 좋을텐데 그럴거 같지는 않네.  

두 코드가 다른 것은 blaze 는 복소수 타입의 연산을 오버로딩해서 구현해놨지만 sph 코드는 c 코드라서 복소수 구조체를 사용하되 연산 자체는  raw하게 되어있다. 

### 09.06

예전 스크립트가 남아있지는 않다. 

음. 로컬 구조체로 연산을 시키고 들어오는 인자를 캐스팅하면 되지않을까? 테스트 해보자. 그러면 iip_sph_pp 의 코드를 그대로 쓸 수 있을 것 같다. 

아 생각해보니까. 데이터가 일차원 배열에 있어야하네. blas 쓸려면.. 데이터 복사 작업을 한번 하든지 아니면 어차피 cpp 코드는 내가 짜야하니까 처음부터 일차원으로 해버리는 것도 괜찮을 것 같다. 성능 차이 한번 테스트는 해봐야겠지만. 

일단은 캐스팅해서 해서 테스트 해보자. 

```matlab
A = [[0.840188 + 0.394383i,0.783099 + 0.798440i],
    [0.911647 + 0.197551i,0.335223 + 0.768230i]];
A
inv(A)
%(-0.795904 + -1.185639i)(1.555857 + 1.099865i)
%(1.588316 + 0.053474i)(-1.528483 + -0.405178i)
```

잘된다. 코드는 iip_sph_pp 의 코드에 캐스팅만 넣어두었다. 

추가하자. 

lapack 에서 work 를 요구하는데 그냥 이거 클래스로 해버릴까? 근데 n을 명시해야하는데. 클래스 생성자에서 받고 루틴을 그냥 호출하는 식으로 일단 해야겠다. 

아 기존의 6by6랑 충돌나네.

refactoring 하고 몇가지 수정해서 넣음

### 09.09

### TODO
+ 파이선 standalone으로 사용할 수 있게하기
+ 런타임에서 쓰레드수를 정하게 하기. 
==> 개발용이 아닌 임의의 pc에서 설치하고 그 pc에 맞게 실행이 가능한가? 의 문제.

임의의 pc에서 gcp 테스트를 해야하는데. 

1. 이 프로젝트를 들고가기
2. 녹음 어플로 녹음 시켜서 별도의 파이썬 코드로 인식  

[윈도콘솔창 없이 런](https://stackoverflow.com/questions/9618815/i-dont-want-console-to-appear-when-i-run-c-program )

### 09.10

#### 전날 발견한 문제

---

시리얼 포트 게인 설정시 응답없음 되는 상황
포트 인식은 됨. 에러도 성공도 아닌 무응답 상태로 빠짐  

---

실행 파일로 release 시킨 상태에서 gcp를 어떻게 쓰게 할 것인가?  
일단 최소한의 요구조건만 맞춘 파이썬을 포함시켜서 실행하였음. 다만 용량이 150mb 정도 되는데 어느 정도까지 줄일 수 있을 지 확인해봐야함.  

---

녹음시 현재는 버튼으로 조작하지만 시간을 설정해서 녹음하는 것도 추가해야할 것 같다.  

---

VAD & GCP 시 vad가 발화를 인식하지 않음.  
데이터가 중간에 없어지는 지 vad 의 문제인지 파악해야함

PreAlgo가 스코프 밖에 있었다. AfterInput 추가하면서 스코프에서 벗어난듯. 수정함.  

그래도 vad 인식은 안된다. 코드가 들어가기는 하는데 어디서 문제난것일까.  

스케일링이 문제인가? 
차이가 없다. 들어오는 입력의 크기에 상관없에 psy가 nan이네.  

vad 생성자를 한번봐야겠다. 생성자는 잘 들어오는데.. 

왜 안되는거냐.. 입력은 잘 들어오는데. vad 알고리즘 코드를 건든 적도 없는데.  좀 더 테스트 해보자. 


### 09.16

wav input 시에 인식은 되는거 같은데 ui부분에서 터짐.

stdio.h에서 오류 발생. windows 라서 발생한 것일 확률이 높다.  

+ GCP 부분에서 문제일으켰을 가능성이 크다. 

----

real time은

raw,data 둘 다 들어오는 데는 문제가 없지만.  

wav input에 비하면 전체적인 크기가 작다. 

wav input 이면 psy가 제대로 들어오는데, real time 이면 nan이 뜨고. 그냥 전체 값을 찍어서 봐야겠다. 

아 스피커 설정 
### 09.02 

### TODO
+ 녹음 기능 추가 - CLI,Recorder,GUI
+ installer 손보기
+ 아이콘 달기
+ 리눅스 인스톨러

Widget에서 integer 인지 아닌지 명시를 하자. 추가함. 

LineEdit에서 변경시 double integer 유효성 확인 후 수정사항 적용시키자. 


TextEdit 시 보다

```
void QLineEdit::textChanged(const QString &text)
This signal is emitted whenever the text changes. The text argument is the new text.

Unlike textEdited(), this signal is also emitted when the text is changed programmatically, for example, by calling setText().

Note: Notifier signal for property text.
```

추가함. 

---

PLASMA 를 써보자.
v19 를 써보려 했는데, omp 4.5 이상을 요구한다.  
v17은 cmake가 없네. 
v18 도 omp 4.5 이상을 요구하네, Window 에서 2.0 버전 그것도 불완전한 2.0 까지만 지원하는데.. 

blas를 필수적으로 까는 거 이전에 여기서 문제가 있을 수 있다. 

ICC를 쓰면 되는데,, ICC 쓰면 verdigris가 안되고.

```
make[2]: *** [CMakeFiles/iip_demo.dir/src/K/KSpectrogram.cpp.o] 오류 2
/home/kbh/git/IIP_Demo/lib/verdigris/wobjectimpl.h(755): error: expression must have a constant value
      constexpr static Arrays arrays = buildArrays();
```

```cpp
  constexpr static auto buildArrays() {
        auto r = Arrays{};
        DataBuilder b{r};
        generateDataPass<T>(b);
        return r;
    }
```

```constexpr``` 이 맞는데. 

선택지
1. 다른 egien 라이브러리 추가 조사
2. verdigris fix
3. eigen library 사용
4. ???

문제점
1. OpenMp > 2.0   < - > MSVC  ==> ICC로 해결
2. Verdigris(GUI) < - > ICC   ==> ???

---

openblas 써보자. 

lapacke.h 를 써야한다. 저번에 어떻게 썼더라? lapacke.h 는 어디서 가져온 거고?

```lapack-netlib/INCLUDE``` 안에 들어는 있는데 이게 맞는 건가? 일단 그거 가져와서 하는데 되는 거 같다. 

문제는 lapack 에 함수 종류가 많네 

```
zheev
zheev_2stage
zheevd
zheevd_2stage
zheevr
zheevr_2stage
zheevx
zheevx_2stage
zhegv
zhegv_2stage
zhegvd
zhegvx
```

다 고윳값을 구한다는 거 같은데, 조금씩 다르다. 


+  ZHEEV
: computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A.

+  ZHEEV_2STAGE
:  computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A using the 2stage technique for
 the reduction to tridiagonal.

+ ZHEEVD 
: computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A.  If eigenvectors are desired, it uses a
 divide and conquer algorithm.
 The divide and conquer algorithm makes very mild assumptions about
 floating point arithmetic. It will work on machines with a guard
 digit in add/subtract, or on those binary machines without guard
 digits which subtract like the Cray X-MP, Cray Y-MP, Cray C-90, or
 Cray-2. It could conceivably fail on hexadecimal or decimal machines
 without guard digits, but we know of none.

+ ZHEEVR
: computes selected eigenvalues and, optionally, eigenvectors
 of a complex Hermitian matrix A.  Eigenvalues and eigenvectors can
 be selected by specifying either a range of values or a range of
 indices for the desired eigenvalues.

 ZHEEVR first reduces the matrix A to tridiagonal form T with a call
 to ZHETRD.  Then, whenever possible, ZHEEVR calls ZSTEMR to compute
 eigenspectrum using Relatively Robust Representations.  ZSTEMR
 computes eigenvalues by the dqds algorithm, while orthogonal
 eigenvectors are computed from various "good" L D L^T representations
 (also known as Relatively Robust Representations). Gram-Schmidt
 orthogonalization is avoided as far as possible. More specifically,
 the various steps of the algorithm are as follows.

 For each unreduced block (submatrix) of T,
    (a) Compute T - sigma I  = L D L^T, so that L and D
        define all the wanted eigenvalues to high relative accuracy.
        This means that small relative changes in the entries of D and L
        cause only small relative changes in the eigenvalues and
        eigenvectors. The standard (unfactored) representation of the
        tridiagonal matrix T does not have this property in general.
    (b) Compute the eigenvalues to suitable accuracy.
        If the eigenvectors are desired, the algorithm attains full
        accuracy of the computed eigenvalues only right before
        the corresponding vectors have to be computed, see steps c) and d).
    (c) For each cluster of close eigenvalues, select a new
        shift close to the cluster, find a new factorization, and refine
        the shifted eigenvalues to suitable accuracy.
    (d) For each eigenvalue with a large enough relative separation compute
        the corresponding eigenvector by forming a rank revealing twisted
        factorization. Go back to (c) for any clusters that remain.

 The desired accuracy of the output can be specified by the input
 parameter ABSTOL.

 For more details, see DSTEMR's documentation and:
 - Inderjit S. Dhillon and Beresford N. Parlett: "Multiple representations
   to compute orthogonal eigenvectors of symmetric tridiagonal matrices,"
   Linear Algebra and its Applications, 387(1), pp. 1-28, August 2004.
 - Inderjit Dhillon and Beresford Parlett: "Orthogonal Eigenvectors and
   Relative Gaps," SIAM Journal on Matrix Analysis and Applications, Vol. 25,
   2004.  Also LAPACK Working Note 154.
 - Inderjit Dhillon: "A new O(n^2) algorithm for the symmetric
   tridiagonal eigenvalue/eigenvector problem",
   Computer Science Division Technical Report No. UCB/CSD-97-971,
   UC Berkeley, May 1997.


 Note 1 : ZHEEVR calls ZSTEMR when the full spectrum is requested
 on machines which conform to the ieee-754 floating point standard.
 ZHEEVR calls DSTEBZ and ZSTEIN on non-ieee machines and
 when partial spectrum requests are made.

 Normal execution of ZSTEMR may create NaNs and infinities and
 hence may abort due to a floating point exception in environments
 which do not handle NaNs and infinities in the ieee standard default
 manner.

+  ZHEEVX
 :  computes selected eigenvalues and, optionally, eigenvectors
 of a complex Hermitian matrix A.  Eigenvalues and eigenvectors can
 be selected by specifying either a range of values or a range of
 indices for the desired eigenvalues.

+  ZHEGV
:  computes all the eigenvalues, and optionally, the eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.
 Here A and B are assumed to be Hermitian and B is also
 positive definite.

+  ZHEGVD 
: computes all the eigenvalues, and optionally, the eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.  Here A and
 B are assumed to be Hermitian and B is also positive definite.
 If eigenvectors are desired, it uses a divide and conquer algorithm.

 The divide and conquer algorithm makes very mild assumptions about
 floating point arithmetic. It will work on machines with a guard
 digit in add/subtract, or on those binary machines without guard
 digits which subtract like the Cray X-MP, Cray Y-MP, Cray C-90, or
 Cray-2. It could conceivably fail on hexadecimal or decimal machines
 without guard digits, but we know of none.

+  ZHEGVX 
 : computes selected eigenvalues, and optionally, eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.  Here A and
 B are assumed to be Hermitian and B is also positive definite.
 Eigenvalues and eigenvectors can be selected by specifying either a
 range of values or a range of indices for the desired eigenvalues.

일단 가장 기본 형태부터 써보자. 


```fortran
subroutine zheev	
(
character 	JOBZ,
character 	UPLO,
integer 	N,
complex*16, dimension( lda, * ) 	A,
integer 	LDA,
double precision, dimension( * ) 	W,
complex*16, dimension( * ) 	WORK,
integer 	LWORK,
double precision, dimension( * ) 	RWORK,
integer 	INFO 
)	
```

```cpp
 LAPACK_zheev( &jobz, &uplo, &n, a, &lda, w, work, &lwork, rwork,
                      &info );
```

```

[in]	JOBZ	
          JOBZ is CHARACTER*1
          = 'N':  Compute eigenvalues only;
          = 'V':  Compute eigenvalues and eigenvectors.


[in]	UPLO	
          UPLO is CHARACTER*1
          = 'U':  Upper triangle of A is stored;
          = 'L':  Lower triangle of A is stored.
[in]	N	
          N is INTEGER
          The order of the matrix A.  N >= 0.
[in,out]	A	
          A is COMPLEX*16 array, dimension (LDA, N)
          On entry, the Hermitian matrix A.  If UPLO = 'U', the
          leading N-by-N upper triangular part of A contains the
          upper triangular part of the matrix A.  If UPLO = 'L',
          the leading N-by-N lower triangular part of A contains
          the lower triangular part of the matrix A.
          On exit, if JOBZ = 'V', then if INFO = 0, A contains the
          orthonormal eigenvectors of the matrix A.
          If JOBZ = 'N', then on exit the lower triangle (if UPLO='L')
          or the upper triangle (if UPLO='U') of A, including the
          diagonal, is destroyed.
[in]	LDA	
          LDA is INTEGER
          The leading dimension of the array A.  LDA >= max(1,N).
[out]	W	
          W is DOUBLE PRECISION array, dimension (N)
          If INFO = 0, the eigenvalues in ascending order.
[out]	WORK	
          WORK is COMPLEX*16 array, dimension (MAX(1,LWORK))
          On exit, if INFO = 0, WORK(1) returns the optimal LWORK.
[in]	LWORK	
          LWORK is INTEGER
          The length of the array WORK.  LWORK >= max(1,2*N-1).
          For optimal efficiency, LWORK >= (NB+1)*N,
          where NB is the blocksize for ZHETRD returned by ILAENV.

          If LWORK = -1, then a workspace query is assumed; the routine
          only calculates the optimal size of the WORK array, returns
          this value as the first entry of the WORK array, and no error
          message related to LWORK is issued by XERBLA.
[out]	RWORK	
          RWORK is DOUBLE PRECISION array, dimension (max(1, 3*N-2))
[out]	INFO	
          INFO is INTEGER
          = 0:  successful exit
          < 0:  if INFO = -i, the i-th argument had an illegal value
          > 0:  if INFO = i, the algorithm failed to converge; i
                off-diagonal elements of an intermediate tridiagonal
                form did not converge to zero.

```

BLAS 에서는 복소수도 double 2개로 받더니, 여긴 또 ```__complex__ double```을 달라고한다. 

일단 캐스팅해서 돌려보자. 돌아가기는 하고, 멀티 쓰레딩도 된다. 

데이터 읽어서 연산은 시켰으나. 검증하는 부분을 구현해야겠다. 

데이터 넣는 부분부터 틀렸다. 
행렬이 두 알고리즘 모두 잘 안들어간다. 

### 09.03

N = 4 가지고 테스트 하자

```matlab
   A
   1.2589 + 0.0000i   1.0763 - 0.5200i   0.1690 - 0.2269i   1.7411 - 0.6080i
   1.0763 + 0.5200i  -1.6098 + 0.0000i   0.4868 - 0.1828i   0.0645 - 1.5256i
   0.1690 + 0.2269i   0.4868 + 0.1828i  -1.3695 + 0.0000i   1.5417 + 0.6276i
   1.7411 + 0.6080i   0.0645 + 1.5256i   1.5417 - 0.6276i  -1.4325 + 0.0000i

   V
   0.2812 + 0.0093i  -0.1539 + 0.0352i  -0.3312 + 0.4423i  -0.6516 + 0.4074i
  -0.1814 - 0.4834i   0.6173 + 0.2624i  -0.2216 - 0.3752i  -0.2401 + 0.1899i
   0.3595 + 0.2247i   0.5886 - 0.3612i   0.5137 + 0.0752i  -0.2598 - 0.0792i
  -0.6889 + 0.0000i  -0.2196 + 0.0000i   0.4850 + 0.0000i  -0.4919 + 0.0000i

   D
   -4.1976         0         0         0
         0   -1.5736         0         0
         0         0   -0.2949         0
         0         0         0    2.9132

   sum(A*V  -  V*D,'all') = -1.8180e-15 + 4.7531e-16i

```

일단 eigen Library 의 A*V - V*D 가 0에 근사함.

---

OpenBLAS의 LAPACK을 해보자. BLAS 출력을 잘못하고 있었다. idx 초기화를 안했네. 

OpenBLAS의 LAPACK이 잘 되는지 검증을 해보자. 

cplx mat * cplx mat  - cplx mat * real vec  을 해야하는데  
행렬 곱은 blas 쓰고 행렬x벡터는 for문 그냥 돌리자.  

zheev 가 inplace 였다. 복사도 해야하네. 
아니다 inplace 시키는게 hermitian 이라 대칭이므로 Upper 또는 Lower에서 넣어준다는 것. 그러면 대칭행렬 곱 연산 루틴이 있다면 그걸 쓰면 되지 않을까 싶은데 생각해보니까 대각성분이 바뀌니까. 안될거 같네.

복사해서 해야겠다. 

cblas하고 lapacke 하고 같이 include 하니까 충돌나는데, 

OpenBLAS 빌드시에 LAPACKE 을 쓴다는 옵션을 줘야했던가 같은 실낱같은 기억이 스쳐지나갔다. ```iip_sph_pp``` 를 진행할 떄는 apt-get 으로 패키지 openblas를 사용했던것 같다 - lapack 포함 빌드인 - 

상당히 꼬일 거 같은데.. linux에서 이걸 해결해서 빌드하고 링크한다 쳐도 윈도우에서는 더 복잡하게 가야하지 않을까? 아니다 cmake에 MSVC 옵션이 있기는 하니까? 근데 얘들 윈도우에서 잘 안되잖아. 

조사를 더 해봐야겠다. 

https://github.com/xianyi/OpenBLAS/wiki/How-to-use-OpenBLAS-in-Microsoft-Visual-Studio  

된다는 거 같다. 

그럼 OpenBLAS가 lapack을 같이 쓸 수 있게하는 옵션을 찾아보자.

일단 OpenBLAS안의 lapack 폴더는 그냥 netlib 의 lapack을 냅다 들고온 거같다. 빌드도 안되네 이건 옵션을 제대로 안줘서 인거 같다.  

아니면 편하게 apt-get으로 한거를 파일 가져와서 링크하며 되지 않을까?  

https://github.com/gogyzzz/iip_sph_pp/issues/91  아.. 좀 더 자세히 읽었어야 했네. 여기서 한 대로 다시 해보자.  

make install 해서 이동된 헤더들만 사용. 빌드는 성공하였다. 이제 다시 테스트로 돌아가자.  

테스트 성공. 

---

Eigen 과 Lapack  모두 오차가 최대 e-14 이다 대체로 e-16 정도 되는 것 같다. 


lapack 이 쓰레딩도 하고 최적화도 되어있어서 10배 빠르다. matlab 과는 6이상일 떄는 많이 격차가 나지만 그 이하에서는 거의 차이나지 않는다. 

ZHEEV_2STAGE 이랑 ZHEEVD 만 해보면 될거 같다. 다른 것들은 같은 알고리즘에 좀 더 특정적인 인터페이스로 보임.  

일단 ZHEEVD 먼저 해보자.  

---

openblas를 submodule 로해서 환경에 맞게 빌드하도록 해야겠다. 설치파일로 쓸거는 어쩔 수 없을 거 같긴한데. 

OpenBLAS에 dynamic으로 하는게 있던게 그것도 한번 알아봐야겠다. 

### 09.04

#### TODO
+ ZHEEV_2STAGE
+ ZHEEVD
+ BLAZE
+ PLASMA - QUARK
+ lapack 래핑. 

---

zheev_2stage 는 OpenBLAS 에서 구현이 안되어있네,

LAPACKE_zheevd 는 zheevd 랑 2stage 둘 다 구현이 되어있다. 

zheevd 가 파라매터가 다르고 integer array work가 추가적으로 필요하다.  

첵크해서 추가할까. 아니면 그냥 사이즈 받을 때 할당 해버릴까. 아니면 상속 구조로 갈까? 

상속 구조로 가는게 나은거 같기도 하네. 

zheev -> zheevd 상속.  

---

zheevd 랑 zheev 가 그렇게 차이가 없다. size 2, 4 일 때 빠른 알고리즘이 서로 바뀌긴하나. 유의미한 차이를 내지 못한다. 

zheevd_2stage 가 ```symbol lookup error: undefined symbol : zheevd_2stage``` 가 뜬다. 구현이 안된걸까. 내가 링크를 잘 못한 걸까. 오타를 낸걸까.  

https://github.com/xianyi/OpenBLAS/blob/ce3651516f12079f3ca2418aa85b9ad571c3a391/lapack-netlib/LAPACKE/src/lapacke_zheevd.c

https://github.com/xianyi/OpenBLAS/blob/ce3651516f12079f3ca2418aa85b9ad571c3a391/lapack-netlib/LAPACKE/src/lapacke_zheevd_2stage.c

보니까 내가 쓰고 있는 함수랑 인자가 다른데..? 뭔가 다른걸 링크해서 쓰고 있는 건가. 뭘 가져다 쓰고 있는 거지.  

---

역행렬 구하는 것 처럼. 고윳값 구하는 것도 코드짜여진거-낮은 채널- 있으면 찾아보자.

일단 blaze 에는 안보이는 거 같다. 

이건 좀 구하기 힘들거 같은데, 

---

### TODO
+ BLAZE
+ PLASMA - QUARK(구버전)
+ (openBLAS를 쓴다면)openblas set-up script 
+ 일반적인 역행렬 함수

blaze 를 일단 써보자. 

blaze is mere wrapping of blas/lapack.  옛날에도 blas에서 사용하려고 해서 알아보니까. 그냥 함수 래핑이었다. 결국엔 어떤 blas를 사용하는가의 문제이지 이게 뭘가를 해줄 거 같지는 않다. 

그럼 일단 이정도 까지만 하자

코드 짠거를 프로젝트에 합치자. 그리고 원래 프로젝트 작업 계속하자. 

BLAS/LAPACK을 처리하는게 급선무겠다. 
https://github.com/kooBH/IIP_Demo/issues/302  


---
### 09.05

### TODO
+ BLAZE
+ PLASMA - QUARK(구버전)
+ (openBLAS를 쓴다면)openblas set-up script 
+ 일반적인 역행렬 함수


역행렬 구하는 루틴을 만들자. 

일단  blaze 에서 가져와서 변환할까 iip_sph_pp 에서 가져와서 변환할까 부터 정하자. 

둘 다 별 차이는 없을 듯. 각자의 타입을 사용하도록 되어있는데. 나는 그냥 double 2개를 re,im 으로 해서 raw하게 할 것인기 떄문에 
어차피 다 변환해야하는 건 같다. 커스텀한 타입을 사용하는 것은 그 타입을 계속 사용한다는 전제하에서는 굉장히 편리하지만, 그렇지 않을 경우 귀찮고 까다로워진다. 그냥 dobule 형으로 해도 직관적으로 받아들이기 쉽기 때문에 이렇게 간다. 

일단 복소수만 하면 되니까 복소수 행렬만 변환하는 스크립트를 짜보자. 전에 짠게 남아있으면 좋을텐데 그럴거 같지는 않네.  

두 코드가 다른 것은 blaze 는 복소수 타입의 연산을 오버로딩해서 구현해놨지만 sph 코드는 c 코드라서 복소수 구조체를 사용하되 연산 자체는  raw하게 되어있다. 

### 09.06

예전 스크립트가 남아있지는 않다. 

음. 로컬 구조체로 연산을 시키고 들어오는 인자를 캐스팅하면 되지않을까? 테스트 해보자. 그러면 iip_sph_pp 의 코드를 그대로 쓸 수 있을 것 같다. 

아 생각해보니까. 데이터가 일차원 배열에 있어야하네. blas 쓸려면.. 데이터 복사 작업을 한번 하든지 아니면 어차피 cpp 코드는 내가 짜야하니까 처음부터 일차원으로 해버리는 것도 괜찮을 것 같다. 성능 차이 한번 테스트는 해봐야겠지만. 

일단은 캐스팅해서 해서 테스트 해보자. 

```matlab
A = [[0.840188 + 0.394383i,0.783099 + 0.798440i],
    [0.911647 + 0.197551i,0.335223 + 0.768230i]];
A
inv(A)
%(-0.795904 + -1.185639i)(1.555857 + 1.099865i)
%(1.588316 + 0.053474i)(-1.528483 + -0.405178i)
```

잘된다. 코드는 iip_sph_pp 의 코드에 캐스팅만 넣어두었다. 

추가하자. 

lapack 에서 work 를 요구하는데 그냥 이거 클래스로 해버릴까? 근데 n을 명시해야하는데. 클래스 생성자에서 받고 루틴을 그냥 호출하는 식으로 일단 해야겠다. 

아 기존의 6by6랑 충돌나네.

refactoring 하고 몇가지 수정해서 넣음

### 09.09

### TODO
+ 파이선 standalone으로 사용할 수 있게하기
+ 런타임에서 쓰레드수를 정하게 하기. 
==> 개발용이 아닌 임의의 pc에서 설치하고 그 pc에 맞게 실행이 가능한가? 의 문제.

임의의 pc에서 gcp 테스트를 해야하는데. 

1. 이 프로젝트를 들고가기
2. 녹음 어플로 녹음 시켜서 별도의 파이썬 코드로 인식  

[윈도콘솔창 없이 런](https://stackoverflow.com/questions/9618815/i-dont-want-console-to-appear-when-i-run-c-program )

### 09.10

#### 전날 발견한 문제

---

시리얼 포트 게인 설정시 응답없음 되는 상황
포트 인식은 됨. 에러도 성공도 아닌 무응답 상태로 빠짐  

---

실행 파일로 release 시킨 상태에서 gcp를 어떻게 쓰게 할 것인가?  
일단 최소한의 요구조건만 맞춘 파이썬을 포함시켜서 실행하였음. 다만 용량이 150mb 정도 되는데 어느 정도까지 줄일 수 있을 지 확인해봐야함.  

---

녹음시 현재는 버튼으로 조작하지만 시간을 설정해서 녹음하는 것도 추가해야할 것 같다.  

---

VAD & GCP 시 vad가 발화를 인식하지 않음.  
데이터가 중간에 없어지는 지 vad 의 문제인지 파악해야함

PreAlgo가 스코프 밖에 있었다. AfterInput 추가하면서 스코프에서 벗어난듯. 수정함.  

그래도 vad 인식은 안된다. 코드가 들어가기는 하는데 어디서 문제난것일까.  

스케일링이 문제인가? 
차이가 없다. 들어오는 입력의 크기에 상관없에 psy가 nan이네.  

vad 생성자를 한번봐야겠다. 생성자는 잘 들어오는데.. 

왜 안되는거냐.. 입력은 잘 들어오는데. vad 알고리즘 코드를 건든 적도 없는데.  좀 더 테스트 해보자. 


### 09.16

wav input 시에 인식은 되는거 같은데 ui부분에서 터짐.

stdio.h에서 오류 발생. windows 라서 발생한 것일 확률이 높다.  

+ GCP 부분에서 문제일으켰을 가능성이 크다. 

----

real time은

raw,data 둘 다 들어오는 데는 문제가 없지만.  

wav input에 비하면 전체적인 크기가 작다. 

wav input 이면 psy가 제대로 들어오는데, real time 이면 nan이 뜨고. 그냥 전체 값을 찍어서 봐야겠다. 

아 스피커 설정 문제였네.  

노트북 48k, 마이크 16k 이건 간과하다니. 삽질이었다. 

덕분에 이것저것 문제점을 찾기는 했다.

### 09.17

잘못된 오디오를 선택해서 construct 한 후에, 다시 제대로된 오디오를 연결하고 reconstruct 시 ``` QObject::disconnect: signal not found in KProcess``` 발생

이거는 

connect 를 체크하는 루틴을 찾아보자. 

QDebug를 쓰라하네. 이 문제는 Module이 할당이 제대로 안된 상태에서 connect를 하면 connect가 안되는데 disconnect를 하려해서 발생하는 에러이다. 

RT input에서 장치 선택시 문제가 있으면 에러를 출력하는데. 이를 bool 로 받아서 이 값을 체크해서 유효한 인스턴스인지 아닌지를 식별해보자. 
이걸 new operator 단에서 어떻게 nullptr을 리턴하게 할 수 있을까?

테스트 해보자.

는 안됨. try - catch 로 필요한 부분만 처리를 해주자 일단. 

지금 RT_INPUT을 생성하다가 문제가 생기면 throw를 하는데. 그러면 module 은 어떻게 되는 거지? 

그 전까지 할당된 메모리들은 어떻게 되는 거지?

일단은 현 시점에서 할당하다가 만 상태로 해제하면 seg 발생. 

다 예외처리 박아버리면 될 거 같기는 한데. 너무 노가다 아닌가. 

다 예외처리 함. 동작잘함.  

---

Process의 UI가 너무 작다. 크기를 늘리고 재배치해보자.  

아니다 그렇게 까지 크게할 필요는 없을거 같은데.  

그렇게 까지 중요한 문제는 아니니까 보류. 

wav plot의 width가 부모 위젯에 의존적이지가 않네. 상대적으로 바뀌게 해야겠다.

---

옵션은 benchmark랑 recorder,dev 만 두고 나머지는 없애자. 

배포판에 포함하는 거는 윈도만 하자. 

GCP 전처리 분기를 해제 하면서 많은 문제가 발생할 거 같은데. 이건 해야하는 일이니까. 

조심스럽게 진행하자. 일단 현재 빌드는 가능. 

아. GCP 옵션이 켜져있을 때만 GCP를 사용하도록 해야겠다. 

RealTimeRecord에 _GCP 구문이 있다?

---

CMakeLists.txt 에서 버전 명시할 수 있게 하자. 
함. 하지만 옮기기만 하고 좀 더 편리하게 할 방법을 찾아보자. 




### 09.18  

아직까지 연구실 네트워크 연결이 안되어있는 관계로 노트북에서 작업 지속.  

pc 리포랑 머지할 때 굉장히 많은 conflict가 날거 같다. 이 상황이 지속될 수 록 많이 나겠지. 

일단 윈도에서의 문제를 해결 하자. 조심스럽게 진행해야한다. 테스크탑에서 수정사항이 좀 광범위하게 있기 때문에.. 

---

일단 stdio.h 에서 발생하는 에러부터 해결해보자. 

```GCP_Module.cpp```

```c++
void GCP::Call(const char* file_name){
#ifndef NDEBUG
  printf("GCP::call %d %s\n",2,file_name);
#endif
  sprintf(command,"%s",file_name);  // <---- stdio.h 에러 발생 지점
}
```

command ```char[80]``` 에서 문제가 발생   

그리고 

```
예외 발생(0x00007FFAC5E41689(vcruntime140.dll), iip_demo.exe): 0xC0000005: 0xFFFFFFFFFFFFFFFF 
위치를 읽는 동안 액세스 위반이 발생했습니다..
```
가
```c++
printf("LOG::%s\n",file_name);
```
했을 때 발생.

file_name 이랑 command 둘 다 클래스의 멤버 변수로 정적 할당되어있는 char 이다. 

어디서 에러가 나는 거지. 

this 가 null 이었다. 이 문제는 데스크탑에서 수정 된 사항인데.. 

코드를 수정하는 거는 좀 지양해야겠다. 

---

최소한의 크기를 차지하고 gcp를 동작시키는 파이썬 배포판을 만들어보자. 안쓰는 패키지들 다 삭제하면 되지 않을까? 

cmake는 데스크탑에서 수정한게 있는데 윈도도 수정되어서 음.. 일단 해봐야겠네. 

---

윈도우 콘솔에 왜 또 한글이 안나오나. 

```인식중``` 이 ```?�식 ��?.``` 으로 나온다. 콘솔도, UI도. 아마 코드상에서 안받는 건가. 

```
DirectWrite: CreateFontFaceFromHDC() failed (글꼴 파일과 같은 입력 파일의 오류를 나타냅니다.) for QFontDef(Family="Fixedsys",
 pointsize=24, pixelsize=20, styleHint=5, weight=50, stretch=100, hintingPreference=0) LOGFONT("Fixedsys", lfWidth=0, 
lfHeight=-20) dpi=120
```

폰트 문제인가 UI에서는 ? 원래 잘 되지 않았나? 

일단 콘솔부터 한글을 출력해보자. 

VS2019는 멀티바이트로 되어있었네. 

아 vs 가 멀티바이트 인코딩이었는 데 이걸 다 유니코드로 바꾸면서 한글 다 깨짐. 이래서 영어로 주석을 달아야..

굉장히 조진거 같다. 원래 유니코드 한글을 잘 표시되는데 멀티바이트 한글은 다 깨짐. 변환이 잘 안되네. 

현재 콘솔 출력창은 한글이 잘 나오는데. 위젯에서 하나도 안된다.

이제 

```
DirectWrite: CreateFontFaceFromHDC() failed (글꼴 파일과 같은 입력 파일의 오류를 나타냅니다.) for QFontDef(Family="Fixedsys", 
pointsize=24, pixelsize=20, styleHint=5, weight=50, stretch=100, hintingPreference=0) LOGFONT("Fixedsys", lfWidth=0, 
lfHeight=-20) dpi=120
```

이 문제를 해결해야하는가. 

일단 머지는 해야겠다. 

했던거 다 날리고 머지하자.

src 폴더 내의 것들만 날리고 머지함. 

폰트문제는 아니네. 별도로 한글 폰트 구해서 넣었는데 안됨. 

```c++
  TE_output->append(QString::fromStdString("한글 QString::fromStdString"));
  TE_output->append(QString("한글 QString"));
  TE_output->append("한글");
```
이게

```
�ѱ� QString::fromStdString
�ѱ� QString
�ѱ�
```

이래 뜬다. 

```c++
 TE_output->append(QString::from("한글 fromLocal8Bit"));
```

으로 출력 성공.. 어째서..

일단은 string -> char* -> QString 이렇게 출력.




---

```cpp
void UI_Module::SendLog(const char* _log){
  printf("SendLog : %s\n",_log);
//  emit(SignalLog( _log ));
  krun->temp_log = std::string(_log);
  emit(SignalLogAlt());
}
```

여기서 로그가 잘 안찍힘. 인자 전달이 잘 안되서 멤버 변수를 직접수정하고 갱신을 요청하는 방식을 사용했는데, 

signal - slot은 별도의 Qt 쓰레드에서 관리하기 때문에 

```
SendLog : 2019-09-18_16-36-08.wav Created
SendLog : 인식중
slot_logAlt 인식중
slot_logAlt 인식중
```

이런 결과가 생긴다. 

일단은 logAlt 대신 log로 다시 돌림. 

현재는 작동 잘하나 지켜봐야함.



---

시리얼 포트 문제를 파악해보자.

---

GCP 안된다. 파이썬이 안열리는가.

열리기는 하는데. 

웨이브 생성후 except 되네.

```python
        config = types.RecognitionConfig(
            encoding=enums.RecognitionConfig.AudioEncoding.LINEAR16,
            sample_rate_hertz=16000,
            language_code='ko-KR')
```

여기서 except  되네. 음.

---

VAD 튜닝이 필요할 것 같다. 노트북 입력일때랑 MEMS 입력일 때랑 확실히 다르네

### 09.19

GCP - speech가 안됨.  잘되던 코드도 안 되는 걸로 보아 계정문제인거 같은데. iipsogang의 비번을 모르겠다. 

엥간한거 다 넣어봤는데 안되네

비번찾음 새로운 까먹은 조합이 있었네. 

```무료 평가판이 종료되었지만 Google Cloud Platform을 계속 사용할 수 있습니다. 서비스를 복원하려면 2019년 10월 17일까지 업그레이드하세요.```

라고 뜨는군 

윈도 경로문제인거 같음. 

경로를 손보니

```python
print('GCP::requesting : '  +speech_file )
response = client.recognize(config, audio)
```
여기서 문제네. recognize를 안해주네.

2018년 9월 17일. 12개월 무료 평가판 기간이 2019년 9월 17일 부로 끝난듯.
결제해야하네

 TODO
+ 녹음 기능 추가 - CLI,Recorder,GUI
+ installer 손보기
+ 아이콘 달기
+ 리눅스 인스톨러

Widget에서 integer 인지 아닌지 명시를 하자. 추가함. 

LineEdit에서 변경시 double integer 유효성 확인 후 수정사항 적용시키자. 


TextEdit 시 보다

```
void QLineEdit::textChanged(const QString &text)
This signal is emitted whenever the text changes. The text argument is the new text.

Unlike textEdited(), this signal is also emitted when the text is changed programmatically, for example, by calling setText().

Note: Notifier signal for property text.
```

추가함. 

---

PLASMA 를 써보자.
v19 를 써보려 했는데, omp 4.5 이상을 요구한다.  
v17은 cmake가 없네. 
v18 도 omp 4.5 이상을 요구하네, Window 에서 2.0 버전 그것도 불완전한 2.0 까지만 지원하는데.. 

blas를 필수적으로 까는 거 이전에 여기서 문제가 있을 수 있다. 

ICC를 쓰면 되는데,, ICC 쓰면 verdigris가 안되고.

```
make[2]: *** [CMakeFiles/iip_demo.dir/src/K/KSpectrogram.cpp.o] 오류 2
/home/kbh/git/IIP_Demo/lib/verdigris/wobjectimpl.h(755): error: expression must have a constant value
      constexpr static Arrays arrays = buildArrays();
```

```cpp
  constexpr static auto buildArrays() {
        auto r = Arrays{};
        DataBuilder b{r};
        generateDataPass<T>(b);
        return r;
    }
```

```constexpr``` 이 맞는데. 

선택지
1. 다른 egien 라이브러리 추가 조사
2. verdigris fix
3. eigen library 사용
4. ???

문제점
1. OpenMp > 2.0   < - > MSVC  ==> ICC로 해결
2. Verdigris(GUI) < - > ICC   ==> ???

---

openblas 써보자. 

lapacke.h 를 써야한다. 저번에 어떻게 썼더라? lapacke.h 는 어디서 가져온 거고?

```lapack-netlib/INCLUDE``` 안에 들어는 있는데 이게 맞는 건가? 일단 그거 가져와서 하는데 되는 거 같다. 

문제는 lapack 에 함수 종류가 많네 

```
zheev
zheev_2stage
zheevd
zheevd_2stage
zheevr
zheevr_2stage
zheevx
zheevx_2stage
zhegv
zhegv_2stage
zhegvd
zhegvx
```

다 고윳값을 구한다는 거 같은데, 조금씩 다르다. 


+  ZHEEV
: computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A.

+  ZHEEV_2STAGE
:  computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A using the 2stage technique for
 the reduction to tridiagonal.

+ ZHEEVD 
: computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A.  If eigenvectors are desired, it uses a
 divide and conquer algorithm.
 The divide and conquer algorithm makes very mild assumptions about
 floating point arithmetic. It will work on machines with a guard
 digit in add/subtract, or on those binary machines without guard
 digits which subtract like the Cray X-MP, Cray Y-MP, Cray C-90, or
 Cray-2. It could conceivably fail on hexadecimal or decimal machines
 without guard digits, but we know of none.

+ ZHEEVR
: computes selected eigenvalues and, optionally, eigenvectors
 of a complex Hermitian matrix A.  Eigenvalues and eigenvectors can
 be selected by specifying either a range of values or a range of
 indices for the desired eigenvalues.

 ZHEEVR first reduces the matrix A to tridiagonal form T with a call
 to ZHETRD.  Then, whenever possible, ZHEEVR calls ZSTEMR to compute
 eigenspectrum using Relatively Robust Representations.  ZSTEMR
 computes eigenvalues by the dqds algorithm, while orthogonal
 eigenvectors are computed from various "good" L D L^T representations
 (also known as Relatively Robust Representations). Gram-Schmidt
 orthogonalization is avoided as far as possible. More specifically,
 the various steps of the algorithm are as follows.

 For each unreduced block (submatrix) of T,
    (a) Compute T - sigma I  = L D L^T, so that L and D
        define all the wanted eigenvalues to high relative accuracy.
        This means that small relative changes in the entries of D and L
        cause only small relative changes in the eigenvalues and
        eigenvectors. The standard (unfactored) representation of the
        tridiagonal matrix T does not have this property in general.
    (b) Compute the eigenvalues to suitable accuracy.
        If the eigenvectors are desired, the algorithm attains full
        accuracy of the computed eigenvalues only right before
        the corresponding vectors have to be computed, see steps c) and d).
    (c) For each cluster of close eigenvalues, select a new
        shift close to the cluster, find a new factorization, and refine
        the shifted eigenvalues to suitable accuracy.
    (d) For each eigenvalue with a large enough relative separation compute
        the corresponding eigenvector by forming a rank revealing twisted
        factorization. Go back to (c) for any clusters that remain.

 The desired accuracy of the output can be specified by the input
 parameter ABSTOL.

 For more details, see DSTEMR's documentation and:
 - Inderjit S. Dhillon and Beresford N. Parlett: "Multiple representations
   to compute orthogonal eigenvectors of symmetric tridiagonal matrices,"
   Linear Algebra and its Applications, 387(1), pp. 1-28, August 2004.
 - Inderjit Dhillon and Beresford Parlett: "Orthogonal Eigenvectors and
   Relative Gaps," SIAM Journal on Matrix Analysis and Applications, Vol. 25,
   2004.  Also LAPACK Working Note 154.
 - Inderjit Dhillon: "A new O(n^2) algorithm for the symmetric
   tridiagonal eigenvalue/eigenvector problem",
   Computer Science Division Technical Report No. UCB/CSD-97-971,
   UC Berkeley, May 1997.


 Note 1 : ZHEEVR calls ZSTEMR when the full spectrum is requested
 on machines which conform to the ieee-754 floating point standard.
 ZHEEVR calls DSTEBZ and ZSTEIN on non-ieee machines and
 when partial spectrum requests are made.

 Normal execution of ZSTEMR may create NaNs and infinities and
 hence may abort due to a floating point exception in environments
 which do not handle NaNs and infinities in the ieee standard default
 manner.

+  ZHEEVX
 :  computes selected eigenvalues and, optionally, eigenvectors
 of a complex Hermitian matrix A.  Eigenvalues and eigenvectors can
 be selected by specifying either a range of values or a range of
 indices for the desired eigenvalues.

+  ZHEGV
:  computes all the eigenvalues, and optionally, the eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.
 Here A and B are assumed to be Hermitian and B is also
 positive definite.

+  ZHEGVD 
: computes all the eigenvalues, and optionally, the eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.  Here A and
 B are assumed to be Hermitian and B is also positive definite.
 If eigenvectors are desired, it uses a divide and conquer algorithm.

 The divide and conquer algorithm makes very mild assumptions about
 floating point arithmetic. It will work on machines with a guard
 digit in add/subtract, or on those binary machines without guard
 digits which subtract like the Cray X-MP, Cray Y-MP, Cray C-90, or
 Cray-2. It could conceivably fail on hexadecimal or decimal machines
 without guard digits, but we know of none.

+  ZHEGVX 
 : computes selected eigenvalues, and optionally, eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.  Here A and
 B are assumed to be Hermitian and B is also positive definite.
 Eigenvalues and eigenvectors can be selected by specifying either a
 range of values or a range of indices for the desired eigenvalues.

일단 가장 기본 형태부터 써보자. 


```fortran
subroutine zheev	
(
character 	JOBZ,
character 	UPLO,
integer 	N,
complex*16, dimension( lda, * ) 	A,
integer 	LDA,
double precision, dimension( * ) 	W,
complex*16, dimension( * ) 	WORK,
integer 	LWORK,
double precision, dimension( * ) 	RWORK,
integer 	INFO 
)	
```

```cpp
 LAPACK_zheev( &jobz, &uplo, &n, a, &lda, w, work, &lwork, rwork,
                      &info );
```

```

[in]	JOBZ	
          JOBZ is CHARACTER*1
          = 'N':  Compute eigenvalues only;
          = 'V':  Compute eigenvalues and eigenvectors.


[in]	UPLO	
          UPLO is CHARACTER*1
          = 'U':  Upper triangle of A is stored;
          = 'L':  Lower triangle of A is stored.
[in]	N	
          N is INTEGER
          The order of the matrix A.  N >= 0.
[in,out]	A	
          A is COMPLEX*16 array, dimension (LDA, N)
          On entry, the Hermitian matrix A.  If UPLO = 'U', the
          leading N-by-N upper triangular part of A contains the
          upper triangular part of the matrix A.  If UPLO = 'L',
          the leading N-by-N lower triangular part of A contains
          the lower triangular part of the matrix A.
          On exit, if JOBZ = 'V', then if INFO = 0, A contains the
          orthonormal eigenvectors of the matrix A.
          If JOBZ = 'N', then on exit the lower triangle (if UPLO='L')
          or the upper triangle (if UPLO='U') of A, including the
          diagonal, is destroyed.
[in]	LDA	
          LDA is INTEGER
          The leading dimension of the array A.  LDA >= max(1,N).
[out]	W	
          W is DOUBLE PRECISION array, dimension (N)
          If INFO = 0, the eigenvalues in ascending order.
[out]	WORK	
          WORK is COMPLEX*16 array, dimension (MAX(1,LWORK))
          On exit, if INFO = 0, WORK(1) returns the optimal LWORK.
[in]	LWORK	
          LWORK is INTEGER
          The length of the array WORK.  LWORK >= max(1,2*N-1).
          For optimal efficiency, LWORK >= (NB+1)*N,
          where NB is the blocksize for ZHETRD returned by ILAENV.

          If LWORK = -1, then a workspace query is assumed; the routine
          only calculates the optimal size of the WORK array, returns
          this value as the first entry of the WORK array, and no error
          message related to LWORK is issued by XERBLA.
[out]	RWORK	
          RWORK is DOUBLE PRECISION array, dimension (max(1, 3*N-2))
[out]	INFO	
          INFO is INTEGER
          = 0:  successful exit
          < 0:  if INFO = -i, the i-th argument had an illegal value
          > 0:  if INFO = i, the algorithm failed to converge; i
                off-diagonal elements of an intermediate tridiagonal
                form did not converge to zero.

```

BLAS 에서는 복소수도 double 2개로 받더니, 여긴 또 ```__complex__ double```을 달라고한다. 

일단 캐스팅해서 돌려보자. 돌아가기는 하고, 멀티 쓰레딩도 된다. 

데이터 읽어서 연산은 시켰으나. 검증하는 부분을 구현해야겠다. 

데이터 넣는 부분부터 틀렸다. 
행렬이 두 알고리즘 모두 잘 안들어간다. 

### 09.03

N = 4 가지고 테스트 하자

```matlab
   A
   1.2589 + 0.0000i   1.0763 - 0.5200i   0.1690 - 0.2269i   1.7411 - 0.6080i
   1.0763 + 0.5200i  -1.6098 + 0.0000i   0.4868 - 0.1828i   0.0645 - 1.5256i
   0.1690 + 0.2269i   0.4868 + 0.1828i  -1.3695 + 0.0000i   1.5417 + 0.6276i
   1.7411 + 0.6080i   0.0645 + 1.5256i   1.5417 - 0.6276i  -1.4325 + 0.0000i

   V
   0.2812 + 0.0093i  -0.1539 + 0.0352i  -0.3312 + 0.4423i  -0.6516 + 0.4074i
  -0.1814 - 0.4834i   0.6173 + 0.2624i  -0.2216 - 0.3752i  -0.2401 + 0.1899i
   0.3595 + 0.2247i   0.5886 - 0.3612i   0.5137 + 0.0752i  -0.2598 - 0.0792i
  -0.6889 + 0.0000i  -0.2196 + 0.0000i   0.4850 + 0.0000i  -0.4919 + 0.0000i

   D
   -4.1976         0         0         0
         0   -1.5736         0         0
         0         0   -0.2949         0
         0         0         0    2.9132

   sum(A*V  -  V*D,'all') = -1.8180e-15 + 4.7531e-16i

```

일단 eigen Library 의 A*V - V*D 가 0에 근사함.

---

OpenBLAS의 LAPACK을 해보자. BLAS 출력을 잘못하고 있었다. idx 초기화를 안했네. 

OpenBLAS의 LAPACK이 잘 되는지 검증을 해보자. 

cplx mat * cplx mat  - cplx mat * real vec  을 해야하는데  
행렬 곱은 blas 쓰고 행렬x벡터는 for문 그냥 돌리자.  

zheev 가 inplace 였다. 복사도 해야하네. 
아니다 inplace 시키는게 hermitian 이라 대칭이므로 Upper 또는 Lower에서 넣어준다는 것. 그러면 대칭행렬 곱 연산 루틴이 있다면 그걸 쓰면 되지 않을까 싶은데 생각해보니까 대각성분이 바뀌니까. 안될거 같네.

복사해서 해야겠다. 

cblas하고 lapacke 하고 같이 include 하니까 충돌나는데, 

OpenBLAS 빌드시에 LAPACKE 을 쓴다는 옵션을 줘야했던가 같은 실낱같은 기억이 스쳐지나갔다. ```iip_sph_pp``` 를 진행할 떄는 apt-get 으로 패키지 openblas를 사용했던것 같다 - lapack 포함 빌드인 - 

상당히 꼬일 거 같은데.. linux에서 이걸 해결해서 빌드하고 링크한다 쳐도 윈도우에서는 더 복잡하게 가야하지 않을까? 아니다 cmake에 MSVC 옵션이 있기는 하니까? 근데 얘들 윈도우에서 잘 안되잖아. 

조사를 더 해봐야겠다. 

https://github.com/xianyi/OpenBLAS/wiki/How-to-use-OpenBLAS-in-Microsoft-Visual-Studio  

된다는 거 같다. 

그럼 OpenBLAS가 lapack을 같이 쓸 수 있게하는 옵션을 찾아보자.

일단 OpenBLAS안의 lapack 폴더는 그냥 netlib 의 lapack을 냅다 들고온 거같다. 빌드도 안되네 이건 옵션을 제대로 안줘서 인거 같다.  

아니면 편하게 apt-get으로 한거를 파일 가져와서 링크하며 되지 않을까?  

https://github.com/gogyzzz/iip_sph_pp/issues/91  아.. 좀 더 자세히 읽었어야 했네. 여기서 한 대로 다시 해보자.  

make install 해서 이동된 헤더들만 사용. 빌드는 성공하였다. 이제 다시 테스트로 돌아가자.  

테스트 성공. 

---

Eigen 과 Lapack  모두 오차가 최대 e-14 이다 대체로 e-16 정도 되는 것 같다. 


lapack 이 쓰레딩도 하고 최적화도 되어있어서 10배 빠르다. matlab 과는 6이상일 떄는 많이 격차가 나지만 그 이하에서는 거의 차이나지 않는다. 

ZHEEV_2STAGE 이랑 ZHEEVD 만 해보면 될거 같다. 다른 것들은 같은 알고리즘에 좀 더 특정적인 인터페이스로 보임.  

일단 ZHEEVD 먼저 해보자.  

---

openblas를 submodule 로해서 환경에 맞게 빌드하도록 해야겠다. 설치파일로 쓸거는 어쩔 수 없을 거 같긴한데. 

OpenBLAS에 dynamic으로 하는게 있던게 그것도 한번 알아봐야겠다. 

### 09.04

#### TODO
+ ZHEEV_2STAGE
+ ZHEEVD
+ BLAZE
+ PLASMA - QUARK
+ lapack 래핑. 

---

zheev_2stage 는 OpenBLAS 에서 구현이 안되어있네,

LAPACKE_zheevd 는 zheevd 랑 2stage 둘 다 구현이 되어있다. 

zheevd 가 파라매터가 다르고 integer array work가 추가적으로 필요하다.  

첵크해서 추가할까. 아니면 그냥 사이즈 받을 때 할당 해버릴까. 아니면 상속 구조로 갈까? 

상속 구조로 가는게 나은거 같기도 하네. 

zheev -> zheevd 상속.  

---

zheevd 랑 zheev 가 그렇게 차이가 없다. size 2, 4 일 때 빠른 알고리즘이 서로 바뀌긴하나. 유의미한 차이를 내지 못한다. 

zheevd_2stage 가 ```symbol lookup error: undefined symbol : zheevd_2stage``` 가 뜬다. 구현이 안된걸까. 내가 링크를 잘 못한 걸까. 오타를 낸걸까.  

https://github.com/xianyi/OpenBLAS/blob/ce3651516f12079f3ca2418aa85b9ad571c3a391/lapack-netlib/LAPACKE/src/lapacke_zheevd.c

https://github.com/xianyi/OpenBLAS/blob/ce3651516f12079f3ca2418aa85b9ad571c3a391/lapack-netlib/LAPACKE/src/lapacke_zheevd_2stage.c

보니까 내가 쓰고 있는 함수랑 인자가 다른데..? 뭔가 다른걸 링크해서 쓰고 있는 건가. 뭘 가져다 쓰고 있는 거지.  

---

역행렬 구하는 것 처럼. 고윳값 구하는 것도 코드짜여진거-낮은 채널- 있으면 찾아보자.

일단 blaze 에는 안보이는 거 같다. 

이건 좀 구하기 힘들거 같은데, 

---

### TODO
+ BLAZE
+ PLASMA - QUARK(구버전)
+ (openBLAS를 쓴다면)openblas set-up script 
+ 일반적인 역행렬 함수

blaze 를 일단 써보자. 

blaze is mere wrapping of blas/lapack.  옛날에도 blas에서 사용하려고 해서 알아보니까. 그냥 함수 래핑이었다. 결국엔 어떤 blas를 사용하는가의 문제이지 이게 뭘가를 해줄 거 같지는 않다. 

그럼 일단 이정도 까지만 하자

코드 짠거를 프로젝트에 합치자. 그리고 원래 프로젝트 작업 계속하자. 

BLAS/LAPACK을 처리하는게 급선무겠다. 
https://github.com/kooBH/IIP_Demo/issues/302  


---
### 09.05

### TODO
+ BLAZE
+ PLASMA - QUARK(구버전)
+ (openBLAS를 쓴다면)openblas set-up script 
+ 일반적인 역행렬 함수


역행렬 구하는 루틴을 만들자. 

일단  blaze 에서 가져와서 변환할까 iip_sph_pp 에서 가져와서 변환할까 부터 정하자. 

둘 다 별 차이는 없을 듯. 각자의 타입을 사용하도록 되어있는데. 나는 그냥 double 2개를 re,im 으로 해서 raw하게 할 것인기 떄문에 
어차피 다 변환해야하는 건 같다. 커스텀한 타입을 사용하는 것은 그 타입을 계속 사용한다는 전제하에서는 굉장히 편리하지만, 그렇지 않을 경우 귀찮고 까다로워진다. 그냥 dobule 형으로 해도 직관적으로 받아들이기 쉽기 때문에 이렇게 간다. 

일단 복소수만 하면 되니까 복소수 행렬만 변환하는 스크립트를 짜보자. 전에 짠게 남아있으면 좋을텐데 그럴거 같지는 않네.  

두 코드가 다른 것은 blaze 는 복소수 타입의 연산을 오버로딩해서 구현해놨지만 sph 코드는 c 코드라서 복소수 구조체를 사용하되 연산 자체는  raw하게 되어있다. 

### 09.06

예전 스크립트가 남아있지는 않다. 

음. 로컬 구조체로 연산을 시키고 들어오는 인자를 캐스팅하면 되지않을까? 테스트 해보자. 그러면 iip_sph_pp 의 코드를 그대로 쓸 수 있을 것 같다. 

아 생각해보니까. 데이터가 일차원 배열에 있어야하네. blas 쓸려면.. 데이터 복사 작업을 한번 하든지 아니면 어차피 cpp 코드는 내가 짜야하니까 처음부터 일차원으로 해버리는 것도 괜찮을 것 같다. 성능 차이 한번 테스트는 해봐야겠지만. 

일단은 캐스팅해서 해서 테스트 해보자. 

```matlab
A = [[0.840188 + 0.394383i,0.783099 + 0.798440i],
    [0.911647 + 0.197551i,0.335223 + 0.768230i]];
A
inv(A)
%(-0.795904 + -1.185639i)(1.555857 + 1.099865i)
%(1.588316 + 0.053474i)(-1.528483 + -0.405178i)
```

잘된다. 코드는 iip_sph_pp 의 코드에 캐스팅만 넣어두었다. 

추가하자. 

lapack 에서 work 를 요구하는데 그냥 이거 클래스로 해버릴까? 근데 n을 명시해야하는데. 클래스 생성자에서 받고 루틴을 그냥 호출하는 식으로 일단 해야겠다. 

아 기존의 6by6랑 충돌나네.

refactoring 하고 몇가지 수정해서 넣음

### 09.09

### TODO
+ 파이선 standalone으로 사용할 수 있게하기
+ 런타임에서 쓰레드수를 정하게 하기. 
==> 개발용이 아닌 임의의 pc에서 설치하고 그 pc에 맞게 실행이 가능한가? 의 문제.

임의의 pc에서 gcp 테스트를 해야하는데. 

1. 이 프로젝트를 들고가기
2. 녹음 어플로 녹음 시켜서 별도의 파이썬 코드로 인식  

[윈도콘솔창 없이 런](https://stackoverflow.com/questions/9618815/i-dont-want-console-to-appear-when-i-run-c-program )

### 09.10

#### 전날 발견한 문제

---

시리얼 포트 게인 설정시 응답없음 되는 상황
포트 인식은 됨. 에러도 성공도 아닌 무응답 상태로 빠짐  

---

실행 파일로 release 시킨 상태에서 gcp를 어떻게 쓰게 할 것인가?  
일단 최소한의 요구조건만 맞춘 파이썬을 포함시켜서 실행하였음. 다만 용량이 150mb 정도 되는데 어느 정도까지 줄일 수 있을 지 확인해봐야함.  

---

녹음시 현재는 버튼으로 조작하지만 시간을 설정해서 녹음하는 것도 추가해야할 것 같다.  

---

VAD & GCP 시 vad가 발화를 인식하지 않음.  
데이터가 중간에 없어지는 지 vad 의 문제인지 파악해야함

PreAlgo가 스코프 밖에 있었다. AfterInput 추가하면서 스코프에서 벗어난듯. 수정함.  

그래도 vad 인식은 안된다. 코드가 들어가기는 하는데 어디서 문제난것일까.  

스케일링이 문제인가? 
차이가 없다. 들어오는 입력의 크기에 상관없에 psy가 nan이네.  

vad 생성자를 한번봐야겠다. 생성자는 잘 들어오는데.. 

왜 안되는거냐.. 입력은 잘 들어오는데. vad 알고리즘 코드를 건든 적도 없는데.  좀 더 테스트 해보자. 


### 09.16

wav input 시에 인식은 되는거 같은데 ui부분에서 터짐.

stdio.h에서 오류 발생. windows 라서 발생한 것일 확률이 높다.  

+ GCP 부분에서 문제일으켰을 가능성이 크다. 

----

real time은

raw,data 둘 다 들어오는 데는 문제가 없지만.  

wav input에 비하면 전체적인 크기가 작다. 

wav input 이면 psy가 제대로 들어오는데, real time 이면 nan이 뜨고. 그냥 전체 값을 찍어서 봐야겠다. 

아 스피커 설정 
### 09.02 

### TODO
+ 녹음 기능 추가 - CLI,Recorder,GUI
+ installer 손보기
+ 아이콘 달기
+ 리눅스 인스톨러

Widget에서 integer 인지 아닌지 명시를 하자. 추가함. 

LineEdit에서 변경시 double integer 유효성 확인 후 수정사항 적용시키자. 


TextEdit 시 보다

```
void QLineEdit::textChanged(const QString &text)
This signal is emitted whenever the text changes. The text argument is the new text.

Unlike textEdited(), this signal is also emitted when the text is changed programmatically, for example, by calling setText().

Note: Notifier signal for property text.
```

추가함. 

---

PLASMA 를 써보자.
v19 를 써보려 했는데, omp 4.5 이상을 요구한다.  
v17은 cmake가 없네. 
v18 도 omp 4.5 이상을 요구하네, Window 에서 2.0 버전 그것도 불완전한 2.0 까지만 지원하는데.. 

blas를 필수적으로 까는 거 이전에 여기서 문제가 있을 수 있다. 

ICC를 쓰면 되는데,, ICC 쓰면 verdigris가 안되고.

```
make[2]: *** [CMakeFiles/iip_demo.dir/src/K/KSpectrogram.cpp.o] 오류 2
/home/kbh/git/IIP_Demo/lib/verdigris/wobjectimpl.h(755): error: expression must have a constant value
      constexpr static Arrays arrays = buildArrays();
```

```cpp
  constexpr static auto buildArrays() {
        auto r = Arrays{};
        DataBuilder b{r};
        generateDataPass<T>(b);
        return r;
    }
```

```constexpr``` 이 맞는데. 

선택지
1. 다른 egien 라이브러리 추가 조사
2. verdigris fix
3. eigen library 사용
4. ???

문제점
1. OpenMp > 2.0   < - > MSVC  ==> ICC로 해결
2. Verdigris(GUI) < - > ICC   ==> ???

---

openblas 써보자. 

lapacke.h 를 써야한다. 저번에 어떻게 썼더라? lapacke.h 는 어디서 가져온 거고?

```lapack-netlib/INCLUDE``` 안에 들어는 있는데 이게 맞는 건가? 일단 그거 가져와서 하는데 되는 거 같다. 

문제는 lapack 에 함수 종류가 많네 

```
zheev
zheev_2stage
zheevd
zheevd_2stage
zheevr
zheevr_2stage
zheevx
zheevx_2stage
zhegv
zhegv_2stage
zhegvd
zhegvx
```

다 고윳값을 구한다는 거 같은데, 조금씩 다르다. 


+  ZHEEV
: computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A.

+  ZHEEV_2STAGE
:  computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A using the 2stage technique for
 the reduction to tridiagonal.

+ ZHEEVD 
: computes all eigenvalues and, optionally, eigenvectors of a
 complex Hermitian matrix A.  If eigenvectors are desired, it uses a
 divide and conquer algorithm.
 The divide and conquer algorithm makes very mild assumptions about
 floating point arithmetic. It will work on machines with a guard
 digit in add/subtract, or on those binary machines without guard
 digits which subtract like the Cray X-MP, Cray Y-MP, Cray C-90, or
 Cray-2. It could conceivably fail on hexadecimal or decimal machines
 without guard digits, but we know of none.

+ ZHEEVR
: computes selected eigenvalues and, optionally, eigenvectors
 of a complex Hermitian matrix A.  Eigenvalues and eigenvectors can
 be selected by specifying either a range of values or a range of
 indices for the desired eigenvalues.

 ZHEEVR first reduces the matrix A to tridiagonal form T with a call
 to ZHETRD.  Then, whenever possible, ZHEEVR calls ZSTEMR to compute
 eigenspectrum using Relatively Robust Representations.  ZSTEMR
 computes eigenvalues by the dqds algorithm, while orthogonal
 eigenvectors are computed from various "good" L D L^T representations
 (also known as Relatively Robust Representations). Gram-Schmidt
 orthogonalization is avoided as far as possible. More specifically,
 the various steps of the algorithm are as follows.

 For each unreduced block (submatrix) of T,
    (a) Compute T - sigma I  = L D L^T, so that L and D
        define all the wanted eigenvalues to high relative accuracy.
        This means that small relative changes in the entries of D and L
        cause only small relative changes in the eigenvalues and
        eigenvectors. The standard (unfactored) representation of the
        tridiagonal matrix T does not have this property in general.
    (b) Compute the eigenvalues to suitable accuracy.
        If the eigenvectors are desired, the algorithm attains full
        accuracy of the computed eigenvalues only right before
        the corresponding vectors have to be computed, see steps c) and d).
    (c) For each cluster of close eigenvalues, select a new
        shift close to the cluster, find a new factorization, and refine
        the shifted eigenvalues to suitable accuracy.
    (d) For each eigenvalue with a large enough relative separation compute
        the corresponding eigenvector by forming a rank revealing twisted
        factorization. Go back to (c) for any clusters that remain.

 The desired accuracy of the output can be specified by the input
 parameter ABSTOL.

 For more details, see DSTEMR's documentation and:
 - Inderjit S. Dhillon and Beresford N. Parlett: "Multiple representations
   to compute orthogonal eigenvectors of symmetric tridiagonal matrices,"
   Linear Algebra and its Applications, 387(1), pp. 1-28, August 2004.
 - Inderjit Dhillon and Beresford Parlett: "Orthogonal Eigenvectors and
   Relative Gaps," SIAM Journal on Matrix Analysis and Applications, Vol. 25,
   2004.  Also LAPACK Working Note 154.
 - Inderjit Dhillon: "A new O(n^2) algorithm for the symmetric
   tridiagonal eigenvalue/eigenvector problem",
   Computer Science Division Technical Report No. UCB/CSD-97-971,
   UC Berkeley, May 1997.


 Note 1 : ZHEEVR calls ZSTEMR when the full spectrum is requested
 on machines which conform to the ieee-754 floating point standard.
 ZHEEVR calls DSTEBZ and ZSTEIN on non-ieee machines and
 when partial spectrum requests are made.

 Normal execution of ZSTEMR may create NaNs and infinities and
 hence may abort due to a floating point exception in environments
 which do not handle NaNs and infinities in the ieee standard default
 manner.

+  ZHEEVX
 :  computes selected eigenvalues and, optionally, eigenvectors
 of a complex Hermitian matrix A.  Eigenvalues and eigenvectors can
 be selected by specifying either a range of values or a range of
 indices for the desired eigenvalues.

+  ZHEGV
:  computes all the eigenvalues, and optionally, the eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.
 Here A and B are assumed to be Hermitian and B is also
 positive definite.

+  ZHEGVD 
: computes all the eigenvalues, and optionally, the eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.  Here A and
 B are assumed to be Hermitian and B is also positive definite.
 If eigenvectors are desired, it uses a divide and conquer algorithm.

 The divide and conquer algorithm makes very mild assumptions about
 floating point arithmetic. It will work on machines with a guard
 digit in add/subtract, or on those binary machines without guard
 digits which subtract like the Cray X-MP, Cray Y-MP, Cray C-90, or
 Cray-2. It could conceivably fail on hexadecimal or decimal machines
 without guard digits, but we know of none.

+  ZHEGVX 
 : computes selected eigenvalues, and optionally, eigenvectors
 of a complex generalized Hermitian-definite eigenproblem, of the form
 A*x=(lambda)*B*x,  A*Bx=(lambda)*x,  or B*A*x=(lambda)*x.  Here A and
 B are assumed to be Hermitian and B is also positive definite.
 Eigenvalues and eigenvectors can be selected by specifying either a
 range of values or a range of indices for the desired eigenvalues.

일단 가장 기본 형태부터 써보자. 


```fortran
subroutine zheev	
(
character 	JOBZ,
character 	UPLO,
integer 	N,
complex*16, dimension( lda, * ) 	A,
integer 	LDA,
double precision, dimension( * ) 	W,
complex*16, dimension( * ) 	WORK,
integer 	LWORK,
double precision, dimension( * ) 	RWORK,
integer 	INFO 
)	
```

```cpp
 LAPACK_zheev( &jobz, &uplo, &n, a, &lda, w, work, &lwork, rwork,
                      &info );
```

```

[in]	JOBZ	
          JOBZ is CHARACTER*1
          = 'N':  Compute eigenvalues only;
          = 'V':  Compute eigenvalues and eigenvectors.


[in]	UPLO	
          UPLO is CHARACTER*1
          = 'U':  Upper triangle of A is stored;
          = 'L':  Lower triangle of A is stored.
[in]	N	
          N is INTEGER
          The order of the matrix A.  N >= 0.
[in,out]	A	
          A is COMPLEX*16 array, dimension (LDA, N)
          On entry, the Hermitian matrix A.  If UPLO = 'U', the
          leading N-by-N upper triangular part of A contains the
          upper triangular part of the matrix A.  If UPLO = 'L',
          the leading N-by-N lower triangular part of A contains
          the lower triangular part of the matrix A.
          On exit, if JOBZ = 'V', then if INFO = 0, A contains the
          orthonormal eigenvectors of the matrix A.
          If JOBZ = 'N', then on exit the lower triangle (if UPLO='L')
          or the upper triangle (if UPLO='U') of A, including the
          diagonal, is destroyed.
[in]	LDA	
          LDA is INTEGER
          The leading dimension of the array A.  LDA >= max(1,N).
[out]	W	
          W is DOUBLE PRECISION array, dimension (N)
          If INFO = 0, the eigenvalues in ascending order.
[out]	WORK	
          WORK is COMPLEX*16 array, dimension (MAX(1,LWORK))
          On exit, if INFO = 0, WORK(1) returns the optimal LWORK.
[in]	LWORK	
          LWORK is INTEGER
          The length of the array WORK.  LWORK >= max(1,2*N-1).
          For optimal efficiency, LWORK >= (NB+1)*N,
          where NB is the blocksize for ZHETRD returned by ILAENV.

          If LWORK = -1, then a workspace query is assumed; the routine
          only calculates the optimal size of the WORK array, returns
          this value as the first entry of the WORK array, and no error
          message related to LWORK is issued by XERBLA.
[out]	RWORK	
          RWORK is DOUBLE PRECISION array, dimension (max(1, 3*N-2))
[out]	INFO	
          INFO is INTEGER
          = 0:  successful exit
          < 0:  if INFO = -i, the i-th argument had an illegal value
          > 0:  if INFO = i, the algorithm failed to converge; i
                off-diagonal elements of an intermediate tridiagonal
                form did not converge to zero.

```

BLAS 에서는 복소수도 double 2개로 받더니, 여긴 또 ```__complex__ double```을 달라고한다. 

일단 캐스팅해서 돌려보자. 돌아가기는 하고, 멀티 쓰레딩도 된다. 

데이터 읽어서 연산은 시켰으나. 검증하는 부분을 구현해야겠다. 

데이터 넣는 부분부터 틀렸다. 
행렬이 두 알고리즘 모두 잘 안들어간다. 

### 09.03

N = 4 가지고 테스트 하자

```matlab
   A
   1.2589 + 0.0000i   1.0763 - 0.5200i   0.1690 - 0.2269i   1.7411 - 0.6080i
   1.0763 + 0.5200i  -1.6098 + 0.0000i   0.4868 - 0.1828i   0.0645 - 1.5256i
   0.1690 + 0.2269i   0.4868 + 0.1828i  -1.3695 + 0.0000i   1.5417 + 0.6276i
   1.7411 + 0.6080i   0.0645 + 1.5256i   1.5417 - 0.6276i  -1.4325 + 0.0000i

   V
   0.2812 + 0.0093i  -0.1539 + 0.0352i  -0.3312 + 0.4423i  -0.6516 + 0.4074i
  -0.1814 - 0.4834i   0.6173 + 0.2624i  -0.2216 - 0.3752i  -0.2401 + 0.1899i
   0.3595 + 0.2247i   0.5886 - 0.3612i   0.5137 + 0.0752i  -0.2598 - 0.0792i
  -0.6889 + 0.0000i  -0.2196 + 0.0000i   0.4850 + 0.0000i  -0.4919 + 0.0000i

   D
   -4.1976         0         0         0
         0   -1.5736         0         0
         0         0   -0.2949         0
         0         0         0    2.9132

   sum(A*V  -  V*D,'all') = -1.8180e-15 + 4.7531e-16i

```

일단 eigen Library 의 A*V - V*D 가 0에 근사함.

---

OpenBLAS의 LAPACK을 해보자. BLAS 출력을 잘못하고 있었다. idx 초기화를 안했네. 

OpenBLAS의 LAPACK이 잘 되는지 검증을 해보자. 

cplx mat * cplx mat  - cplx mat * real vec  을 해야하는데  
행렬 곱은 blas 쓰고 행렬x벡터는 for문 그냥 돌리자.  

zheev 가 inplace 였다. 복사도 해야하네. 
아니다 inplace 시키는게 hermitian 이라 대칭이므로 Upper 또는 Lower에서 넣어준다는 것. 그러면 대칭행렬 곱 연산 루틴이 있다면 그걸 쓰면 되지 않을까 싶은데 생각해보니까 대각성분이 바뀌니까. 안될거 같네.

복사해서 해야겠다. 

cblas하고 lapacke 하고 같이 include 하니까 충돌나는데, 

OpenBLAS 빌드시에 LAPACKE 을 쓴다는 옵션을 줘야했던가 같은 실낱같은 기억이 스쳐지나갔다. ```iip_sph_pp``` 를 진행할 떄는 apt-get 으로 패키지 openblas를 사용했던것 같다 - lapack 포함 빌드인 - 

상당히 꼬일 거 같은데.. linux에서 이걸 해결해서 빌드하고 링크한다 쳐도 윈도우에서는 더 복잡하게 가야하지 않을까? 아니다 cmake에 MSVC 옵션이 있기는 하니까? 근데 얘들 윈도우에서 잘 안되잖아. 

조사를 더 해봐야겠다. 

https://github.com/xianyi/OpenBLAS/wiki/How-to-use-OpenBLAS-in-Microsoft-Visual-Studio  

된다는 거 같다. 

그럼 OpenBLAS가 lapack을 같이 쓸 수 있게하는 옵션을 찾아보자.

일단 OpenBLAS안의 lapack 폴더는 그냥 netlib 의 lapack을 냅다 들고온 거같다. 빌드도 안되네 이건 옵션을 제대로 안줘서 인거 같다.  

아니면 편하게 apt-get으로 한거를 파일 가져와서 링크하며 되지 않을까?  

https://github.com/gogyzzz/iip_sph_pp/issues/91  아.. 좀 더 자세히 읽었어야 했네. 여기서 한 대로 다시 해보자.  

make install 해서 이동된 헤더들만 사용. 빌드는 성공하였다. 이제 다시 테스트로 돌아가자.  

테스트 성공. 

---

Eigen 과 Lapack  모두 오차가 최대 e-14 이다 대체로 e-16 정도 되는 것 같다. 


lapack 이 쓰레딩도 하고 최적화도 되어있어서 10배 빠르다. matlab 과는 6이상일 떄는 많이 격차가 나지만 그 이하에서는 거의 차이나지 않는다. 

ZHEEV_2STAGE 이랑 ZHEEVD 만 해보면 될거 같다. 다른 것들은 같은 알고리즘에 좀 더 특정적인 인터페이스로 보임.  

일단 ZHEEVD 먼저 해보자.  

---

openblas를 submodule 로해서 환경에 맞게 빌드하도록 해야겠다. 설치파일로 쓸거는 어쩔 수 없을 거 같긴한데. 

OpenBLAS에 dynamic으로 하는게 있던게 그것도 한번 알아봐야겠다. 

### 09.04

#### TODO
+ ZHEEV_2STAGE
+ ZHEEVD
+ BLAZE
+ PLASMA - QUARK
+ lapack 래핑. 

---

zheev_2stage 는 OpenBLAS 에서 구현이 안되어있네,

LAPACKE_zheevd 는 zheevd 랑 2stage 둘 다 구현이 되어있다. 

zheevd 가 파라매터가 다르고 integer array work가 추가적으로 필요하다.  

첵크해서 추가할까. 아니면 그냥 사이즈 받을 때 할당 해버릴까. 아니면 상속 구조로 갈까? 

상속 구조로 가는게 나은거 같기도 하네. 

zheev -> zheevd 상속.  

---

zheevd 랑 zheev 가 그렇게 차이가 없다. size 2, 4 일 때 빠른 알고리즘이 서로 바뀌긴하나. 유의미한 차이를 내지 못한다. 

zheevd_2stage 가 ```symbol lookup error: undefined symbol : zheevd_2stage``` 가 뜬다. 구현이 안된걸까. 내가 링크를 잘 못한 걸까. 오타를 낸걸까.  

https://github.com/xianyi/OpenBLAS/blob/ce3651516f12079f3ca2418aa85b9ad571c3a391/lapack-netlib/LAPACKE/src/lapacke_zheevd.c

https://github.com/xianyi/OpenBLAS/blob/ce3651516f12079f3ca2418aa85b9ad571c3a391/lapack-netlib/LAPACKE/src/lapacke_zheevd_2stage.c

보니까 내가 쓰고 있는 함수랑 인자가 다른데..? 뭔가 다른걸 링크해서 쓰고 있는 건가. 뭘 가져다 쓰고 있는 거지.  

---

역행렬 구하는 것 처럼. 고윳값 구하는 것도 코드짜여진거-낮은 채널- 있으면 찾아보자.

일단 blaze 에는 안보이는 거 같다. 

이건 좀 구하기 힘들거 같은데, 

---

### TODO
+ BLAZE
+ PLASMA - QUARK(구버전)
+ (openBLAS를 쓴다면)openblas set-up script 
+ 일반적인 역행렬 함수

blaze 를 일단 써보자. 

blaze is mere wrapping of blas/lapack.  옛날에도 blas에서 사용하려고 해서 알아보니까. 그냥 함수 래핑이었다. 결국엔 어떤 blas를 사용하는가의 문제이지 이게 뭘가를 해줄 거 같지는 않다. 

그럼 일단 이정도 까지만 하자

코드 짠거를 프로젝트에 합치자. 그리고 원래 프로젝트 작업 계속하자. 

BLAS/LAPACK을 처리하는게 급선무겠다. 
https://github.com/kooBH/IIP_Demo/issues/302  


---
### 09.05

### TODO
+ BLAZE
+ PLASMA - QUARK(구버전)
+ (openBLAS를 쓴다면)openblas set-up script 
+ 일반적인 역행렬 함수


역행렬 구하는 루틴을 만들자. 

일단  blaze 에서 가져와서 변환할까 iip_sph_pp 에서 가져와서 변환할까 부터 정하자. 

둘 다 별 차이는 없을 듯. 각자의 타입을 사용하도록 되어있는데. 나는 그냥 double 2개를 re,im 으로 해서 raw하게 할 것인기 떄문에 
어차피 다 변환해야하는 건 같다. 커스텀한 타입을 사용하는 것은 그 타입을 계속 사용한다는 전제하에서는 굉장히 편리하지만, 그렇지 않을 경우 귀찮고 까다로워진다. 그냥 dobule 형으로 해도 직관적으로 받아들이기 쉽기 때문에 이렇게 간다. 

일단 복소수만 하면 되니까 복소수 행렬만 변환하는 스크립트를 짜보자. 전에 짠게 남아있으면 좋을텐데 그럴거 같지는 않네.  

두 코드가 다른 것은 blaze 는 복소수 타입의 연산을 오버로딩해서 구현해놨지만 sph 코드는 c 코드라서 복소수 구조체를 사용하되 연산 자체는  raw하게 되어있다. 

### 09.06

예전 스크립트가 남아있지는 않다. 

음. 로컬 구조체로 연산을 시키고 들어오는 인자를 캐스팅하면 되지않을까? 테스트 해보자. 그러면 iip_sph_pp 의 코드를 그대로 쓸 수 있을 것 같다. 

아 생각해보니까. 데이터가 일차원 배열에 있어야하네. blas 쓸려면.. 데이터 복사 작업을 한번 하든지 아니면 어차피 cpp 코드는 내가 짜야하니까 처음부터 일차원으로 해버리는 것도 괜찮을 것 같다. 성능 차이 한번 테스트는 해봐야겠지만. 

일단은 캐스팅해서 해서 테스트 해보자. 

```matlab
A = [[0.840188 + 0.394383i,0.783099 + 0.798440i],
    [0.911647 + 0.197551i,0.335223 + 0.768230i]];
A
inv(A)
%(-0.795904 + -1.185639i)(1.555857 + 1.099865i)
%(1.588316 + 0.053474i)(-1.528483 + -0.405178i)
```

잘된다. 코드는 iip_sph_pp 의 코드에 캐스팅만 넣어두었다. 

추가하자. 

lapack 에서 work 를 요구하는데 그냥 이거 클래스로 해버릴까? 근데 n을 명시해야하는데. 클래스 생성자에서 받고 루틴을 그냥 호출하는 식으로 일단 해야겠다. 

아 기존의 6by6랑 충돌나네.

refactoring 하고 몇가지 수정해서 넣음

### 09.09

### TODO
+ 파이선 standalone으로 사용할 수 있게하기
+ 런타임에서 쓰레드수를 정하게 하기. 
==> 개발용이 아닌 임의의 pc에서 설치하고 그 pc에 맞게 실행이 가능한가? 의 문제.

임의의 pc에서 gcp 테스트를 해야하는데. 

1. 이 프로젝트를 들고가기
2. 녹음 어플로 녹음 시켜서 별도의 파이썬 코드로 인식  

[윈도콘솔창 없이 런](https://stackoverflow.com/questions/9618815/i-dont-want-console-to-appear-when-i-run-c-program )

### 09.10

#### 전날 발견한 문제

---

시리얼 포트 게인 설정시 응답없음 되는 상황
포트 인식은 됨. 에러도 성공도 아닌 무응답 상태로 빠짐  

---

실행 파일로 release 시킨 상태에서 gcp를 어떻게 쓰게 할 것인가?  
일단 최소한의 요구조건만 맞춘 파이썬을 포함시켜서 실행하였음. 다만 용량이 150mb 정도 되는데 어느 정도까지 줄일 수 있을 지 확인해봐야함.  

---

녹음시 현재는 버튼으로 조작하지만 시간을 설정해서 녹음하는 것도 추가해야할 것 같다.  

---

VAD & GCP 시 vad가 발화를 인식하지 않음.  
데이터가 중간에 없어지는 지 vad 의 문제인지 파악해야함

PreAlgo가 스코프 밖에 있었다. AfterInput 추가하면서 스코프에서 벗어난듯. 수정함.  

그래도 vad 인식은 안된다. 코드가 들어가기는 하는데 어디서 문제난것일까.  

스케일링이 문제인가? 
차이가 없다. 들어오는 입력의 크기에 상관없에 psy가 nan이네.  

vad 생성자를 한번봐야겠다. 생성자는 잘 들어오는데.. 

왜 안되는거냐.. 입력은 잘 들어오는데. vad 알고리즘 코드를 건든 적도 없는데.  좀 더 테스트 해보자. 


### 09.16

wav input 시에 인식은 되는거 같은데 ui부분에서 터짐.

stdio.h에서 오류 발생. windows 라서 발생한 것일 확률이 높다.  

+ GCP 부분에서 문제일으켰을 가능성이 크다. 

----

real time은

raw,data 둘 다 들어오는 데는 문제가 없지만.  

wav input에 비하면 전체적인 크기가 작다. 

wav input 이면 psy가 제대로 들어오는데, real time 이면 nan이 뜨고. 그냥 전체 값을 찍어서 봐야겠다. 

아 스피커 설정 문제였네.  

노트북 48k, 마이크 16k 이건 간과하다니. 삽질이었다. 

덕분에 이것저것 문제점을 찾기는 했다.

### 09.17

잘못된 오디오를 선택해서 construct 한 후에, 다시 제대로된 오디오를 연결하고 reconstruct 시 ``` QObject::disconnect: signal not found in KProcess``` 발생

이거는 

connect 를 체크하는 루틴을 찾아보자. 

QDebug를 쓰라하네. 이 문제는 Module이 할당이 제대로 안된 상태에서 connect를 하면 connect가 안되는데 disconnect를 하려해서 발생하는 에러이다. 

RT input에서 장치 선택시 문제가 있으면 에러를 출력하는데. 이를 bool 로 받아서 이 값을 체크해서 유효한 인스턴스인지 아닌지를 식별해보자. 
이걸 new operator 단에서 어떻게 nullptr을 리턴하게 할 수 있을까?

테스트 해보자.

는 안됨. try - catch 로 필요한 부분만 처리를 해주자 일단. 

지금 RT_INPUT을 생성하다가 문제가 생기면 throw를 하는데. 그러면 module 은 어떻게 되는 거지? 

그 전까지 할당된 메모리들은 어떻게 되는 거지?

일단은 현 시점에서 할당하다가 만 상태로 해제하면 seg 발생. 

다 예외처리 박아버리면 될 거 같기는 한데. 너무 노가다 아닌가. 

다 예외처리 함. 동작잘함.  

---

Process의 UI가 너무 작다. 크기를 늘리고 재배치해보자.  

아니다 그렇게 까지 크게할 필요는 없을거 같은데.  

그렇게 까지 중요한 문제는 아니니까 보류. 

wav plot의 width가 부모 위젯에 의존적이지가 않네. 상대적으로 바뀌게 해야겠다.

---

옵션은 benchmark랑 recorder,dev 만 두고 나머지는 없애자. 

배포판에 포함하는 거는 윈도만 하자. 

GCP 전처리 분기를 해제 하면서 많은 문제가 발생할 거 같은데. 이건 해야하는 일이니까. 

조심스럽게 진행하자. 일단 현재 빌드는 가능. 

아. GCP 옵션이 켜져있을 때만 GCP를 사용하도록 해야겠다. 

RealTimeRecord에 _GCP 구문이 있다?

---

CMakeLists.txt 에서 버전 명시할 수 있게 하자. 
함. 하지만 옮기기만 하고 좀 더 편리하게 할 방법을 찾아보자. 




### 09.18  

아직까지 연구실 네트워크 연결이 안되어있는 관계로 노트북에서 작업 지속.  

pc 리포랑 머지할 때 굉장히 많은 conflict가 날거 같다. 이 상황이 지속될 수 록 많이 나겠지. 

일단 윈도에서의 문제를 해결 하자. 조심스럽게 진행해야한다. 테스크탑에서 수정사항이 좀 광범위하게 있기 때문에.. 

---

일단 stdio.h 에서 발생하는 에러부터 해결해보자. 

```GCP_Module.cpp```

```c++
void GCP::Call(const char* file_name){
#ifndef NDEBUG
  printf("GCP::call %d %s\n",2,file_name);
#endif
  sprintf(command,"%s",file_name);  // <---- stdio.h 에러 발생 지점
}
```

command ```char[80]``` 에서 문제가 발생   

그리고 

```
예외 발생(0x00007FFAC5E41689(vcruntime140.dll), iip_demo.exe): 0xC0000005: 0xFFFFFFFFFFFFFFFF 
위치를 읽는 동안 액세스 위반이 발생했습니다..
```
가
```c++
printf("LOG::%s\n",file_name);
```
했을 때 발생.

file_name 이랑 command 둘 다 클래스의 멤버 변수로 정적 할당되어있는 char 이다. 

어디서 에러가 나는 거지. 

this 가 null 이었다. 이 문제는 데스크탑에서 수정 된 사항인데.. 

코드를 수정하는 거는 좀 지양해야겠다. 

---

최소한의 크기를 차지하고 gcp를 동작시키는 파이썬 배포판을 만들어보자. 안쓰는 패키지들 다 삭제하면 되지 않을까? 

cmake는 데스크탑에서 수정한게 있는데 윈도도 수정되어서 음.. 일단 해봐야겠네. 

---

윈도우 콘솔에 왜 또 한글이 안나오나. 

```인식중``` 이 ```?�식 ��?.``` 으로 나온다. 콘솔도, UI도. 아마 코드상에서 안받는 건가. 

```
DirectWrite: CreateFontFaceFromHDC() failed (글꼴 파일과 같은 입력 파일의 오류를 나타냅니다.) for QFontDef(Family="Fixedsys",
 pointsize=24, pixelsize=20, styleHint=5, weight=50, stretch=100, hintingPreference=0) LOGFONT("Fixedsys", lfWidth=0, 
lfHeight=-20) dpi=120
```

폰트 문제인가 UI에서는 ? 원래 잘 되지 않았나? 

일단 콘솔부터 한글을 출력해보자. 

VS2019는 멀티바이트로 되어있었네. 

아 vs 가 멀티바이트 인코딩이었는 데 이걸 다 유니코드로 바꾸면서 한글 다 깨짐. 이래서 영어로 주석을 달아야..

굉장히 조진거 같다. 원래 유니코드 한글을 잘 표시되는데 멀티바이트 한글은 다 깨짐. 변환이 잘 안되네. 

현재 콘솔 출력창은 한글이 잘 나오는데. 위젯에서 하나도 안된다.

이제 

```
DirectWrite: CreateFontFaceFromHDC() failed (글꼴 파일과 같은 입력 파일의 오류를 나타냅니다.) for QFontDef(Family="Fixedsys", 
pointsize=24, pixelsize=20, styleHint=5, weight=50, stretch=100, hintingPreference=0) LOGFONT("Fixedsys", lfWidth=0, 
lfHeight=-20) dpi=120
```

이 문제를 해결해야하는가. 

일단 머지는 해야겠다. 

했던거 다 날리고 머지하자.

src 폴더 내의 것들만 날리고 머지함. 

폰트문제는 아니네. 별도로 한글 폰트 구해서 넣었는데 안됨. 

```c++
  TE_output->append(QString::fromStdString("한글 QString::fromStdString"));
  TE_output->append(QString("한글 QString"));
  TE_output->append("한글");
```
이게

```
�ѱ� QString::fromStdString
�ѱ� QString
�ѱ�
```

이래 뜬다. 

```c++
 TE_output->append(QString::from("한글 fromLocal8Bit"));
```

으로 출력 성공.. 어째서..

일단은 string -> char* -> QString 이렇게 출력.




---

```cpp
void UI_Module::SendLog(const char* _log){
  printf("SendLog : %s\n",_log);
//  emit(SignalLog( _log ));
  krun->temp_log = std::string(_log);
  emit(SignalLogAlt());
}
```

여기서 로그가 잘 안찍힘. 인자 전달이 잘 안되서 멤버 변수를 직접수정하고 갱신을 요청하는 방식을 사용했는데, 

signal - slot은 별도의 Qt 쓰레드에서 관리하기 때문에 

```
SendLog : 2019-09-18_16-36-08.wav Created
SendLog : 인식중
slot_logAlt 인식중
slot_logAlt 인식중
```

이런 결과가 생긴다. 

일단은 logAlt 대신 log로 다시 돌림. 

현재는 작동 잘하나 지켜봐야함.



---

시리얼 포트 문제를 파악해보자.

---

GCP 안된다. 파이썬이 안열리는가.

열리기는 하는데. 

웨이브 생성후 except 되네.

```python
        config = types.RecognitionConfig(
            encoding=enums.RecognitionConfig.AudioEncoding.LINEAR16,
            sample_rate_hertz=16000,
            language_code='ko-KR')
```

여기서 except  되네. 음.

---

VAD 튜닝이 필요할 것 같다. 노트북 입력일때랑 MEMS 입력일 때랑 확실히 다르네

### 09.19

GCP - speech가 안됨.  잘되던 코드도 안 되는 걸로 보아 계정문제인거 같은데. iipsogang의 비번을 모르겠다. 

엥간한거 다 넣어봤는데 안되네

비번찾음 새로운 까먹은 조합이 있었네. 

```무료 평가판이 종료되었지만 Google Cloud Platform을 계속 사용할 수 있습니다. 서비스를 복원하려면 2019년 10월 17일까지 업그레이드하세요.```

라고 뜨는군 

윈도 경로문제인거 같음. 

경로를 손보니

```python
print('GCP::requesting : '  +speech_file )
response = client.recognize(config, audio)
```
여기서 문제네. recognize를 안해주네.

2018년 9월 17일. 12개월 무료 평가판 기간이 2019년 9월 17일 부로 끝난듯.
결제해야하네

sogangsap 계정에 2019년 9월 19일 부로 12개월  300달라 평가판 시작해서 api 등록함. 
키도 변경함. 

---

VAD 파라매터를 잘 못 잡겠다. 기존의 값으로는 MEMS로는 잘 되는데. 웨이브파일을 읽거나  노트북으로 녹음할 때는 아예 안되네.  


---

slog_logAlt 없앰. 


---

speex

https://www.speex.org/downloads/

GNU 오픈 소스 오디오 포맷. 

vs2008 win32까지만 솔루션이 들어있네. 윈도는 거의 개발 안하는듯.

https://en.wikipedia.org/wiki/Speex
```Xiph.Org now considers Speex obsolete; its successor is the more modern Opus codec, which surpasses its performance in all areas.```

---

```
2019-09-19_15-56-27.wav Created
인식중
2 3 4
2019-09-19_15-56-35.wav Created
인식중
섎굹 
2019-09-19_15-56-49.wav Created
인식중
쒓源⑥덇퉴
```

하 인코딩. 

```
2019-09-19 15:56 : 하나 둘 셋 넷
2019-09-19 15:57 : 유리 깨지니까
2019-09-19 15:57 :  한글이 깨지니까
```

파이썬 단에서는 문제 없는거 같은데, 주고 받는데서 깨지는듯. 

GCP_Module.cpp 의

```c++
   msg = std::string(PyUnicode_AsUTF8(pValue));
```
을 알맞게 바꿔야겠다. 

https://docs.python.org/3/c-api/unicode.html

올바른 함수를 찾아보자. 

이전에도 인코딩때문에 고생했는데. 그냥 별거 안했는데 해결되어서 구체적인 해결책이 없네.  

일단 python 단에서는 str - 유니코드 를 리턴한다. 그렇다면 ```PyUnicode_AsUTF8``` 에서 PyUnicode 부분은 잘 될것이란것.  
그러면 문제는 UTF8이란 건데 
```c++
	QString temp = QString::fromUtf8(PyUnicode_AsUTF8(pValue));
	std::cout << "fromQt::" << temp.toStdString() << "\n";
```
이게 안돌아가는 이유를 알 수가 없네. 

### 09.20

한글 문자열 이슈 계속 처리중. 

https://stackoverflow.com/questions/45575863/how-to-print-utf-8-strings-to-stdcout-on-windows

파이썬에서 utf8으로 제대로 보내주지만 c++에서 string 타입이 utf-8을 안해주는것 같다. 

VS2019가 어느순간인가 멀티바이트 프로젝트로 되어있었다. 

유니코드로 바꿨지만 차이는 없다.

확실한 것은
+ 파이썬은 utf8 인코딩을 사용한다.
+ C에서 받은 이진데이터는 const char*에 저장된 utf8인코딩이다.

이러면 왜 QString 에서 fromUtf8으로 변환했을 때 같은 잘못된 출력을 내보내는지 모르겠다. 

```
The encoded version utf-8": b'\xed\x95\x9c\xea\xb8\x80'
The encoded version cp949: b'\xc7\xd1\xb1\xdb'
```

3-byte utf-8 문자열을 c++에서 처리해야한다. 

http://blog.daum.net/_blog/BlogTypeView.do?blogid=0aP73&articleno=23&_bloghome_menu=recenttext

OS별로 분기를 넣어서 처리해야할거 같은데. 일단 윈도부터 하자. 

wstring을 쓴다해도 2바이트 문자열이기 때문에 3바이트를 쓰는 한글이랑 그렇게 잘 맞지는 않는다.  
지금 해야하는 거는 char* 로 저장된 3바이트 utf8 문자열을 QString으로 받는 것이다. 

```
�ν��� <-fromUtf8
인식중    <-fromLocal8bit
한글      <-fromUtf8
쒓        <-fromLocal8bit
한글      <-fromUtf8
쒓        <-fromLocal8bit
``` 

+ 파이썬에서 받아온 문자열은 utf8이므로 fromUtf8
+ c++ 코드에서 직접 입력된 한글은  formLocal8bit, 이것도 유니코드인데 왜 다르게 해야하는지는 잘 모르겠네. 비주얼 스튜디오에서 어떻게 인식하는 가의 문제인거 같다. 

일단 이렇게 윈도로 처리하고 리눅스는 그때가서 봐야겠네.

한글 잘 나오는 것 확인함.



---


vad 인식도 좀 이상함

```
Module Initialized
2019-09-20_11-52-16.wav Created
인식중


2019-09-20_11-52-17.wav Created
인식중
쒓컯숆탳 吏μ젙蹂爾곌뎄ㅼ엯덈떎
쒓컯숆탳 吏μ젙蹂爾곌뎄ㅼ엯덈떎
2019-09-20_11-52-19.wav Created
인식중
쒓컯숆탳 吏μ젙蹂泥섎━ 寃껋엯덈떎
쒓컯숆탳 吏μ젙蹂泥섎━ 寃껋엯덈떎
```

처음에 돌때는 인식안되다가 한번 더 돌아야 인식한다. 

---

출력 단 항상 새로운 출력을 볼 수 있게 하자. 
```c++
  QObject::connect(TE_output, &QTextEdit::textChanged,
	  [&](){
	  TE_output->moveCursor(QTextCursor::End);
	}
  );
```



---

실시간 vad & GCP 시 한번하고 끝나버린다. 

어디서 멎어버리는데 어디서 멎는지 알아보자. 

input의 stock이 128에서 고정되네.

rt가 stop된다. 

ui module 수정하면서 scope가 잘못되었고 delay를 너무 크게줘서 이런 일이 발생한듯. 수정함. 잘돌아간다

---

얼마전에 시리얼포트 게인 조절할 때 문제가 있었는데. 해당 상황을 다시 발생시켜서  
알아보자. 

```c++
 for (int i = 1; i < 9; i++) {
        data_write[i] = Gain_Value_Table[ch_gain[i]];
        ...

```

여기서 에러

찍어보니까 i 가 0이네. 이러면 안되지. 

ch_gain[]의 default 값이 없어서인듯. 넣어줌. 

### 09.23

app의 샘플레이트가 윈도우 시스템에서 설정한 샘플레이트보다 높을 경우 차이만큼 녹음이 안되는 문제를 해결해보자. 

https://stackoverflow.com/questions/36128507/changing-sample-rate-for-microphone-speakers-in-windows-7

https://social.msdn.microsoft.com/Forums/windowsdesktop/en-US/e2e9858f-9bf5-4400-8a4e-570fa3285aad/changing-audio-device-sample-rate-via-pkeyaudioenginedeviceformat-does-not-work?forum=windowssdk

MEMS 보드의 기본 샘플레이트가 8k라서 문제가 있음.  

https://stackoverflow.com/questions/22616924/wasapi-choosing-a-wave-format-for-exclusive-output

IMMDevice 를 알아보자.  

https://docs.microsoft.com/en-us/windows/win32/api/mmdeviceapi/nn-mmdeviceapi-immdevice


https://stackoverflow.com/questions/46947529/setting-default-format-on-capture-device-via-winapi

MMDevice가 있는데, WASAPI에서도 설정할 수 있을까? 

https://docs.microsoft.com/en-us/windows/win32/coreaudio/enumerating-audio-devices
```
After selecting a suitable device, the client can call the IMMDevice::Activate method to activate the device-specific 
interfaces in WASAPI, the DeviceTopology API, and the EndpointVolume API.
```

둘 다 같이 봐야겠네. 

그런데. 이건 default format인데. default라면 바꿀 수 있다는 것이 아닌가?? 그럼문제는 rt audio오단에서 바껴진 형식이 적용이 안된다는 것인가.

rtaudio 에서는 
```c++
 IMMDeviceEnumerator* deviceEnumerator_;
```
이정도로만 사용하고 설정하는 것은 없다. 

일단 Enumerator 부터 해보자. 

https://docs.microsoft.com/en-us/windows/win32/coreaudio/enumerating-audio-devices

이건 그냥 다 출력한다. 
https://github.com/mvaneerde/blog/blob/develop/audioendpoints/audioendpoints/endpoints.cpp  
  
rt_aduio의 enumerator가 코드 가 더 길지만 출력물이 정제되어 있다. 
https://github.com/thestk/rtaudio/blob/1cba5c90a35b0e79915dc46dd5525da2285a211b/RtAudio.cpp

일단 rt aduio의 코드를 공부해보자. 

```
mfapi.h header
This header is used by Microsoft Media Foundation. For more information, see:

Microsoft Media Foundation mfapi.h contains the following programming interfaces:
```

이런 걸 사용하네. 이거는 입출력 다룰 때 쓰는 거 같으니 pass. 

테스트 프로젝트 작성 중.

---

spectrogram load 시 종료 현상있음. 

---

### 09.24 

https://github.com/gogyzzz/arrayfire_wpe 한번 테스트 해봐야한다.  

https://arrayfire.com/ 랑 CUDA를 쓰는데 둘다 구버전에서 해서 최신버전으로 하면 안될가능성이 크다고 한다.   

아. 일단 nasal을 쓰려고했는데. 이게 상태가 안좋아서 좀 봐야겠는데.  

cochlea 로 터전을 옮길까. 일단 cochlea에서하고 nasal은 시간이 좀 걸릴 거 같다. 

docker 권한이 없네, group 에는 포함이 되어 있는데 . 

도커 19를 써야하는데 cochlea의 ubuntu 16.04에는 18이 깔려있네. 새로 설치를 해야하는데. 그러려면 이전에 도커 쓰던 분들에게 얘기를 해야겠지.

일단 기존의 도커부터 날려보자 

```sudo apt-get remove docker docker-engine docker.io containerd runc```

에 아무것도 걸리는게 없네. ```docker-ce``` 는 걸리는데 이걸로 된다면 그냥 업그레이드하면 되는거 아닐까? 

일단 진행하자 . ???
```sudo apt-get update``` 에서

```

Err:14 https://nvidia.github.io/libnvidia-container/ubuntu16.04/amd64  InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6ED91CA3AC1160CD
Err:19 https://nvidia.github.io/nvidia-container-runtime/ubuntu16.04/amd64  InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6ED91CA3AC1160CD
Err:20 https://nvidia.github.io/nvidia-docker/ubuntu16.04/amd64  InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6ED91CA3AC1160CD
Fetched 1,590 kB in 1s (986 kB/s)
Reading package lists... Done
W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://nvidia.github.io/libnvidia-container/ubuntu16.04/amd64  InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6ED91CA3AC1160CD
W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://nvidia.github.io/nvidia-container-runtime/ubuntu16.04/amd64  InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6ED91CA3AC1160CD
W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://nvidia.github.io/nvidia-docker/ubuntu16.04/amd64  InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6ED91CA3AC1160CD
W: Failed to fetch https://nvidia.github.io/libnvidia-container/ubuntu16.04/amd64/InRelease  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6ED91CA3AC1160CD
W: Failed to fetch https://nvidia.github.io/nvidia-container-runtime/ubuntu16.04/amd64/InRelease  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6ED91CA3AC1160CD
W: Failed to fetch https://nvidia.github.io/nvidia-docker/ubuntu16.04/amd64/InRelease  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6ED91CA3AC1160CD
W: Some index files failed to download. They have been ignored, or old ones used instead.

```
이런 에러가 나네..

그래도 일단 하라는거 하나씩 해보자 

```
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

```

```curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -```
```sudo apt-get install docker-ce docker-ce-cli containerd.io```

```apt-cache madison docker-ce```

해서 나오는 것 중에서 19.03.2~3 을 설치해보자.  

음.. 뒤에다 ubuntu-bionic이라 적혀있는데. bionic은 ubuntu 18이다. 설치하면 안될거 같은데. 

xenial 붙은건 안보이고.. 

```
Install from a package
If you cannot use Docker’s repository to install Docker Engine - Community, you can download the .deb file for your release and install it manually. You need to download a new file each time you want to upgrade Docker.

Go to https://download.docker.com/linux/ubuntu/dists/, choose your Ubuntu version, browse to pool/stable/, choose amd64, armhf, arm64, ppc64el, or s390x, and download the .deb file for the Docker Engine - Community version you want to install.

```
https://download.docker.com/linux/ubuntu/dists/xenial/pool/stable/amd64/ 여기에 있네. 

일단 

https://download.docker.com/linux/ubuntu/dists/xenial/pool/stable/amd64/docker-ce_19.03.2~3-0~ubuntu-xenial_amd64.deb
를 설치해봄 

```docker --version ```
하니 19.03.2가 뜨네. 

```docker run --gpus all tts``` 하니까
``` could not select device driver "" with capabilities: [[gpu]].``` 
음..

https://github.com/NVIDIA/nvidia-docker/issues/1034

http://collabnix.com/introducing-new-docker-cli-api-support-for-nvidia-gpus-under-docker-engine-19-03-0-beta-release/
```sudo ubuntu-drivers autoinstall```

reboot 을 해야하는데. 누가 접속 중이다.   

reboot 해도 그대로다. 아니 

nvidia-smi 는 잘되는거보면 cuda는 잘 깔려있는데. docker가 문제인가. 

버전도 nasal의 버전이 더 낮은데 드라이버는 . nasal은 설치된 docker도 없다. 날라갔나.

```
If you didn't already make sure you've installed the nvidia-container-toolkit.
If this doesn't fix it for you, make sure you've restarted docker systemctl restart dockerd
```
안됨. 

아. 여기 뭔가 있네

```
Troubleshooting:
Did you encounter the below error message:

$ docker run -it --rm --gpus all debian
docker: Error response from daemon: linux runtime spec devices: could not select device driver "" with capabilities: [[gpu]].
The above error means that Nvidia could not properly register with Docker. What it actually mean is the drivers are not properly installed on the host. This could also mean the nvidia container tools were installed without restarting the docker daemon: you need to restart the docker daemon.

I suggest you to go back and verify if nvidia-container-runtime is installed or not OR restart the Docker daemon.
```


도커 데몬을 재시작하니까 
```
docker: Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused 
"process_linux.go:402: container init caused \"process_linux.go:385: running prestart hook 1 caused \\\"error running 
hook: exit status 1, stdout: , stderr: exec command: [/usr/bin/nvidia-container-cli --load-kmods configure --
ldconfig=@/sbin/ldconfig.real --device=all --compute --utility --require=cuda>=9.0 --pid=8276 
/var/lib/docker/overlay2/b956d7f169cca157457e107ee8c99a050c33199ded8f4fa4d68e3ace612c6d0c/merged]\\\\nnvidia-contain
er-cli: initialization error: driver error: failed to process request\\\\n\\\"\"": unknown.

```

 이런 에러를 맞이함 

https://github.com/NVIDIA/nvidia-docker/issues/726

버전이 안 맞는다는 거네. 

--- 



MMdevice로 
읽단 읽는 건 돌려보았다. 설정하는 방법을 알아야한다. 

---

MAC OSX 에 GUI를 달자. 

경로 관련해서 문제가 있을 거 같은데. 일단 돌려는 봐야지. 

Qt5.12.3 을 사용하고 있는 걸로 기억하는데.. 맞겠지?  

LTS 였나 Latest release 였나.. 일단 엥간해서는 default로 가자. 

다운로드가 너무 느리네 하루종일 켜놔야겠다.  

### 09.25

TTS가 안되네. fmt_type : 1 로 받는데 3 이어야하는데. output 폴더가 없어서 다운로드가 안되었다. 작동확인함. 

---

OSX 용 Qt5의 라이브러리 파일이 어디있는 지를 못찾겠다. 

확장자가 없는 unix 이진파일 ```QtCore``` 랑 ```QtCore_debug``` 가 있는데 이게 so 일까? 

https://codejamming.org/2018/deploy-to-macos

MacOS의 동적 라이브러리 사용은 많이 다른것 같다. 

cmake 명령어로 링크해보고 여기서 값을 봐서 해버리자. 

https://github.com/kooBH/CMake-Qt5-MacOS 이거 파서 하기로함. 

find_package로 빌드 성공. 어디서 가져오는지 찾아보자. 

일단 급하니까 스탠드 얼론이 아닌걸로 만들자. 그냥. 

OSX 빌드하니까 에러가 좀 생기네. 

``` use 'template' keyword to treat 'get' as a dependent template name ``` ??


JsonConfig.h 의
```c++
 data[*it] = j[name][*it].get<T>(); //를
 data[*it] = j[name][*it].template get<T>(); //로
```

했는데 mac에서는 gui Recorder를 빌드 시켰다. 다른 os에선 어떻게 될지는 모르겠다. 
일단 폰트나 여러가지것들이 안 맞으니까 다듬자.  

갑자기 링크에서 에러가 나네?  serial_port 관련 부분인거 같은데. 폰트 추가 말고는 한게 없는데. 
폰트문제는 아닌거 같고.. 뭐지. 

macos의 라이브러리들이 안잡히네.  

```CMake
  list(APPEND REC_LIB ${FOUNDATION_LIBRARY} ${IOKIT_LIBRARY} )
```
여기서 안 잡힌다. 

```CMake
if(APPLE)
    find_library(IOKIT_LIBRARY IOKit)
    find_library(FOUNDATION_LIBRARY Foundation)
endif()
```
이게 CMake 분기때문에 날라갔네. 
수정함.  

JSON 파일위치가 별도로 되어있다. 같은 파일을 쓰게하자

3개의 라벨만 검은 글씨인데 나머지는 흰색 글씨이다. 

macos의 기본 글씨색이 흰색으로 되어있는 건가. 다른 os에서는 검은 글씨인데.  

같은 label 도 어떤건 검은색이고 어떤건 흰색이고.. 뭔 차이지. 


---

recorder에 spectrogram 넣어도 괜찮을꺼 같은데. 

---

시간 정해서 녹음하는 기능을 추가하자. 


---

### 09.26

MacOS 폰트 이슈 계속. 

QApplication 자체의 style을 지정하는 방식으로 default text color를 설정할 수 있지 않을까? 

```c++
app.setStyleSheet("color:black;"); 
```
은 안되나
```Could not parse application stylesheet```

전체적으로 적용하는 것은 안되고 


```c++
app.setStyleSheet("QLabel{color:black;}"); 
```
이런 식으로 하면 모든 QLabel에 적용은 된다.

```c++
app.setStyleSheet("\
QLabel{color:black;}\
QPushButton{color:black;}\
"); 
```
이런 식으로 처리함. 

---

macOS에서 장치 이름 깨지는걸 고쳐보자. 

RtAudio에서 ```마이크```,```스피커``` 라는 한글을 인식 못해서 인거 같다. 

MacOS의 텍스트 인코딩이 뭐지? 

hex를 까서 봤는데. 이상한데. 

앞쪽은 동일한데 뒤쪽이 ```ffffffb8```,```ffffffb6``` 이런 식이다. 4바이트 양식인가? 

그러기엔 ```ffffffc7``` 이 2번 반복되는 부분이 있는데. 이건 말아 안된다. 
8바이트당 한 문자인가?  

rtaudio단에서 잘 못받는 걸 수도 있으니까. 나중에 살펴봐야겠다. 

일단은 기능 추가부터 하자. 

---

시간설정해서 녹음하는 기능을 추가하자. RT_Input을 손봐야겠지. 

동작을 어떻게 해야하나. 

RT_Input의 구조를 좀 바꿔야하네. 

```c++
  const unsigned int max_stock_second = 10;
  bool record_inf;
  double record_time;
```
를 추가하고. 

```c++
#if __INF
  data.totalFrames = max_stock;
#else
  data.totalFrames = (unsigned long)(sample_rate * RECORD_TIME); // * dtime
#endif
```

```int rt_call_back(void * /*outputBuffer*/, void *inputBuffer,
                 unsigned int nBufferFrames, double /*streamTime*/,
                 RtAudioStreamStatus status, void *data)```
콜백 함수에서 결정되네. 글로벌로 해둘까. InputData에 담아두자. 

max_stock 과 totalframes를 구분해야한다. 
구분시키고 ```Clear()``` 함수 추가함.

QLineEdit에 녹음 시간을 설정하게 하자. double만 받도록.

작동은 잘하는데 끝나고 종료를 해버리네?

정수형이 되는데 소숫점이 있으면 종료된다. 

 rec_thread->join(); 여기서 터지는디 

```c++
 if ( data.stock.load() > max_stock)
```
이 부분이 stock이 음수인 경우 참으로 들어간다?
```c++
 if (data.stock.load() >0 && data.stock.load() > max_stock)
```
일단 이렇게 때웠다. 

녹음기능은 추가됨. 

아. max_stock < reocrd_time 인 경우를 처리하지 않았네. 
이 경우에는 제3의 procedure를 짜야한다. 

---

style로 다 ```color:black``` 을 박아버려서. disabled 된 것이 구분이 안된다. 

---

테스트 해보려했는데. 샘플레이트가 이상한데. 모든 경우의 수를 다 보여주네. 

### 09.27

파이썬 모듈은 의존성이 있는 데. 옵션으로 줘야하나?? 

---

max_stock 을 넘는 길이로 녹음할 때, read_offset이 링버퍼로 돌아가도록 해야한다. 

구현함. 근데 녹음한 소리가 좀 끊긴다. 샘플레이트 수치는 괜찮은데. 

입력이 일정하게 들어가지 않는다. 계속 발생하니까 Rewind문제는 아닌거 같고. 

다른 녹음프로그램도 같은 문제가 발생한다. 드라이버 문제인가. 디지털 멀티채널 드라이버가 설치되어있는데 지금 아날로그로 작동시키고 있다. 

일단 지금 상태로 MacOS에서 테스트 해보자. 맥 마이크로는 녹음 잘함. 

녹음이 되긴되는데 15sec 하는거보면 rewind문제는 아닌거 같은데 seg가 발생한다. 너무오래 잡아서 그런가? 

16채널이라 문제가 생기는 건가? 

1채널일 때는 문제가 없다. 


16채널일 떄,

```c++
  memcpy(raw, data.buffer + read_offset, temp1 * channels * sizeof(short));
	  temp2 = shift_size - temp1;
	  /* */
	  memcpy(raw + temp1*channels, data.buffer, temp2 * channels * sizeof(short));
	  read_offset = temp2;
```

여기서 temp1 값이 깨지네. 연산을 잘 못하고 있었다. 수정함. 잘 동아간다. 
---

현재 타이머 된 녹음시 프로세스를 잡아서 사용하고 있음. 잡지 않게 하자. 

이렇게 하려면 rt가 끝났을 때. gui에서 그걸 알아내서 마무리 처리를 해야한다. 상호참조를 해야하는데.. 
그냥 qt에서 계속 체크하게 할까? 그러면 의존성 문제는 덜해지는데. .. 그렇게 하자. 

타이머도 달아보자. .
QElapsedTimer를 쓰면 될거 같다. 

QTimer를 쓰려면 slot을 써야하네, recorder에 verdigris를 넣었던가? 

타이머 오차를 좀 손봐야겠다. 

위젯에 스타일 적용이 안된다. 코드상의 변경은 verdigris 밖에 없는데. 

---

disabled 구분 안되는건 확실히 고쳐야겠다.. 

```
QPushButton:disabled {
color:gray;}
```

이런게 있네. 잘 돌아감. 

---

recorder 끝내고 코드 정리를 해야겠다. 너무 지저분하네. 

### 09.30  

https://github.com/kooBH/IIP_Demo/issues/312 
해결하자. 

저 현상이 정확이 어떨때 발생하는지 명시해두지 않아서 제대로 됐는지 모르겠ㄴ. 

실험결과 int 가 음수일때 unsigned int 양수랑 비교하면 음수인 int 가 더 크게 된다. 아마 캐스팅을 unsigned int로 해주는 듯. 
static_cast로 해결 

```c++
 if (data.stock.load() > static_cast<int>(max_stock)) {
```

---

프로젝트에 녹음 모듈도 넣자. 넣음

---

스타일 문제 해결. 위젯의 background-color를 설정했을 때. 하위 위젯의 배경색이 바뀐다. 그안에 별도의 구분이 없더라도  
위젯의 child 위젯을 만들어줘야 스타일 적용이 된다. 


---

https://sourceforge.net/projects/openblas/files/v0.3.6/ 

여기에 

dll 파일과 lib 파일이 다 있는데 사용이 가능한건가? 이거는 MinGW 용인거 같다. 

libgcc_s_seh-1.dll을 요구한다.  

https://stackoverflow.com/questions/2529770/how-to-use-libraries-compiled-with-mingw-in-msvc  
https://stackoverflow.com/questions/29139425/how-to-use-libraries-compiled-with-mingw-in-visual-studio 

MSVC에서 쓰는게 불가능하지는 않지만 어렵고 잘 안되는것 같다. 

예전에 테스트도 했었는데. 

https://github.com/xianyi/OpenBLAS/wiki/How-to-use-OpenBLAS-in-Microsoft-Visual-Studio

로 하나씩 해보자. 

모든 명령어를 miniconda prompt 안에서 수행해야한다. 

빌드하는데 10분 넘게 걸릴거 같다. 

30분걸렸네. openblas.lib : 70MB , 출력물 하나. 

내일 동작하는지 테스트 해보자.











