>#1 블룸 필터란 무엇인가

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424363-05494e10-e230-45b1-9a2a-18f413748970.png)

블룸 필터는 데이터 블록에 특정 key 값의 존재 여부를 빠르게 확인할 수 있는 확률적 자료 구조이다.

Write를 할 때엔 각각의 key에 k개의 해시 함수를 수행하여 각 key 마다 k개의 해시 값을 얻고,

해당 값에 해당하는 블룸 필터 배열 칸의 값을 0에서 1로 수정하여

해당 데이터 블록에 해당 key 값이 존재함을 나타낸다.

반대로 Read를 할 때엔 읽으려는 key에 똑같은 k개의 해시 함수를 수행하여 k개의 해시 값을 얻고

해당 배열 값이 전부 1인 데이터 블록만을 선별하여 읽는 것으로 읽기 증폭을 대폭 감소시킨다.
  
   
  
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424577-84d02a8f-c306-431f-b7b9-df92d3c2027b.png)

참고로 하나의 sstable엔 n개의 데이터 블록과 1개의 필터 블록, 1개의 메타 인덱스 블록이 존재하며

필터 블록엔 n개의 블룸 필터 배열이, 메타 인덱스 블록은 각 블룸 필터 배열이 어느 데이터 블록의 필터인지 저장한다.
  
   
   
   
  
  
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424609-121954ba-8a0d-486e-83f5-c4102a669c8e.png)

블룸 필터의 장점은 True Negative가 발생하지 않는단 점이다.

블룸 필터를 통해 특정 key의 값이 존재하지 않는다 판단되면,

해당 key 값은 절대로 해당 데이터 블록에 존재하지 않는다.

이는 필터의 신뢰성과 직결된 문제이므로 True Negative가 발생하지 않는단 점은 큰 장점이다.
  
   

다만 반대로 존재하지 않는 값을 존재한다 판단하는 False Positive의 경우 종종 발생할 수 있는데,

이는 True Negative보단 덜 중요한 문제이지만

False Positive로 인해 읽을 필요 없는 데이터 블록까지 읽어 성능이 떨어지므로

False Positive를 줄이는 것 역시 블룸 필터의 중요한 과제이다.
  
   
 
<br/>
<br/>
<br/>
<br/>

>#2 블룸 필터의 구조와 해시 함수

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424697-ef93e101-a865-47a3-9e14-2046590dd9d9.png)

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424752-f19bab99-09cd-4c83-9686-c0de4776c88d.png)

db_bench를 통해 실험을 돌릴 때,

우리는 key당 bits 값 (Bloom_bits)이나 쓰거나 읽을 데이터의 갯수 (Num)를 조절할 수 있다.

이때 생성되는 블룸 필터 배열의 크기는 B*N bits가 된다.

이는 블룸필터의 핵심 코드 파일인 bloom.cc의 CreateFilter 클래스 함수에서 확인할 수 있다.

 

db_bench에선 블룸 필터를 사용하지 않는 것이 디폴트 값이기에 Bloom_bits 값을 지정하여 블룸 필터를 켜주어야 하며,

만약 블룸 필터를 사용한다면 LevelDB에서 이상적으로 생각하는 Bloom_bits 값은 10이다.

(이는 쓰기 성능을 떨어뜨리지 않으면서, 읽기 성능을 향상시킬 수 있는 마지노선의 수치이다.)

 


 

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424934-cea90e5b-ca24-449a-90f4-53dc6452f539.png)

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424941-231cf55b-6188-42a0-88ec-8b7c46d644f0.png)

또한 해시 함수의 개수 K의 경우 K = ln2 * (M/N) = ln2 * B이란 수식을 사용한다.

이것은 수학적으로 증명된, 블룸 필터의 false positive를 최소한으로 줄일 수 있는 값이다.

이에 관한 더 자세한 증명은 https://d2.naver.com/helloworld/749531 해당 글을 참조.

이 역시 bloom.cc 파일의 코드에서 구현되어 있는 것을 확인할 수 있다.

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424987-d8a58b47-0e68-4f13-b3d1-bc7ea413406a.png)


false positive가 발생할 확률을 수학적으로 정리하면 위와 같으며,

이를 e의 정의를 통해 정리하면 오른쪽과 같은 식이 나온다.

여기서 k의 값이 ln2 * (m/n)이라면 해당 식은 (1/2)^k의 꼴로 정리되므로

k의 값이 클수록 false positive가 발생할 확률이 작아짐을 알 수 있다.

(그리고 k = ln2 * b = 0.69 b 이므로 b가 증가하면 k가 증가한다.)



![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425038-94b8594d-a3c9-4325-8ddc-61f6c90c89b9.png)

이를 db_bench를 통해 실험해 본 결과 bloom_bits 값이 커질수록

false positive가 적게 발생하여 평균보다 성능이 빠른 경우가 더 많이 발생함을 알 수 있다.

 

 


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425050-e20cbdb2-f988-45aa-bb71-f423c77236f8.png)

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425059-75c0aa55-ee7a-4b32-9905-e97275d20966.png)



다만 bloom bits가 너무 커질경우 오히려 false positive가 더 많이 발생하는 경향을 보였는데,

이는 bloom.cc 코드에서 해시 함수의 최대 개수를 30개로 제한하고 있기에

k = ln2 * b 라는 공식이 지켜지지 않아 되려 false positive가 증가한 것으로 추정된다.

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425778-ecfbef99-a13b-4efb-82ad-c666be9dc22e.png)


해시 함수 갯수 제한을 제거하고 다시 db_bench를 돌려 본 결과,

모든 read 측정값이 평균과 비슷한 값이 나옴을 알 수 있다.

이는 false positive의 발생 확률이 대폭 낮아져서 false positive가 거의 발생하지 않았기 때문으로 추정된다.

허나 이 경우엔 한번의 read에 690번의 해시를 처리해야 하므로 (전자의 경우엔 30번)

false positive 자체는 덜 발생하였으나 이를 위해 read를 처리하는데 필요한 시간은 오히려 증가하였음을 알 수 있다.

 

 <br/>
<br/>
<br/>
<br/>

>#3 블룸 필터의 코드 흐름

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425832-0ca0a6f6-a8d4-471b-8765-44a3ccad1904.png)


LevelDB 전체 코드엔 약 100개 가량의 메인 함수가 존재한다.

여기서 db_bench의 makefile을 확인해 보면 db_bench의 메인 함수는

db_bench.cc 파일에 존재함을 알 수 있다.

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425880-7bcff039-5faa-4e42-9ad9-763c12f6c614.png)


db_bench.cc 파일의 메인 함수는 크게 2개의 부분으로 구성되는데,

첫번째는 db_bench를 실행할때 bloom_bits, num과 같은 파라미터를 읽어들이는 sscanf 파트,

그리고 benchmark.Run()이란 클래스 함수 수행 두 파트로 구성된다.

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425930-e7017bb8-d77e-428a-a4c1-fa9e1b5288ca.png)

benchmark.Run()도 크게 3가지 부분으로 나눌 수 있는데,


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425988-2d5638f4-ba97-4a54-9f5e-ddc3529dbe7a.png)

벤치마크 클래스에서 filter policy라는 클래스 변수가 선언되며

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425998-87fd0957-3bc3-4c7f-9239-ebf597053cf7.png)

이후 벤치마크 생성자에서 filter policy의 내부를 채운다.

메인함수에서 sscanf로 읽은 bloom_bits 값이 0 이상이면

블룸 필터를 사용한다는 것이니 NewBloomFilterPolicy 함수를 호출하고,

아닐 경우엔 블룸 필터를 사용하지 않는다는 것이니 Null을 리턴한다.

(bloom_bits의 디폴트 값은 -1로 따로 설정하지 않으면 블룸 필터를 사용하지 않는다.)

 

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426008-acd92e3d-9adf-4d7b-8986-2c248e6705b1.png)

NewBloomFilterPolicy()는 FilterPolicy 클래스를 오버라이드한 BloomFilterPolicy 클래스를 생성하여 리턴한다.

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426122-bd927d23-6f3e-45f1-a2d4-056b08589f74.png)


https://github.com/google/leveldb/blob/main/include/leveldb/filter_policy.h
 

FilterPolicy 클래스는

필터의 이름을 리턴하는 Name(),

Write할 때 블룸 필터 배열을 생성하는 CreateFilter(),

Read할 때 배열에 특정 key 값이 존재하는지 확인하는 KeyMayMatch() ,

3개의 클래스 함수로 구성되어있다.

이를 오버라이드한 BloomFilterPolicy 클래스의 코드를 살펴보자면

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426214-c4c7fdf2-881d-46b7-80a4-01c4988f520b.png)


https://github.com/google/leveldb/blob/main/util/bloom.cc
해시 함수의 갯수를 정하고 최대 갯수를 제한하는 코드 부분과

이름을 리턴하는 Name() 함수


 ![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426303-d9c4be90-2d6b-4025-8035-5483070159f5.png)

그리고 CreateFilter 함수는 bloom_bits(bits_per_key)값과 num(n)값을 곱해

블룸 필터 배열의 크기(bits)를 정하는 파트와


 ![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426325-b0586409-deeb-4856-b1f2-8611ce6b46e9.png)

"더블 해싱"을 처리하는 파트로 나뉜다.

 

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426381-db205ffc-c946-49af-a75d-2b0869145737.png)
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426385-eb7ead05-1359-4802-9bc6-0f2d892756bb.png)


https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf

해당 함수의 주석에 작성되어 있는 논문을 참고하면,

하나의 key에 k개의 해시 함수를 사용하는 대신 2개의 해시 함수를, 그 중에서 하나는 중복 사용하는

"더블 해싱"을 통해 기존의 블룸 필터와 비슷한 false positive 확률을 유지한 채 

해시 함수로 인한 부하와 연산을 대폭 줄일 수 있음을 수학적으로 증명하고있다.

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426427-2a7c8496-b118-42aa-b6da-112205d9b581.png)

LevelDB에서 첫번째 해시 함수는 BloomHash()란 함수로,

두번째 해시 함수는 ( h>>17 ) | ( h<<15 ) 란 비트 연산으로 구현되어있다.

 


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426431-9ec3e7cb-ba41-48da-b90a-280456b4ceb7.png)
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426436-93fc5dbf-b93a-4b9d-8abc-9b2a41aba6cc.png)


BloomHash는 key 값을 인자로 특정 값을 리턴하는 전형적인 해시함수의 형태를 취하고 있으며,

 


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426489-099b0b6f-8a1c-4263-a575-60407a8b8c65.png)


쉬프트 연산은 앞의 15자리와 뒤의 17자리를 바꾸는 형태로 구성되어있다.

(참고로 해시값은 uint32_t라는 32비트 자료형을 사용한다.)

이러한 쉬프트 연산은 input 값에 따라 output 값이 정해져있으나

input 값이 유사해도 전혀 다른 output 값이 나온다는 해시 함수의 조건을 만족하면서도

필요한 오버헤드가 가장 적어서 사용되는 것으로 추정된다.

 

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426517-b9b43ff8-3541-474a-8549-2132edf178f5.png)



그리고 특이사항으론 hash 하나당 하나의 bit를 1로 바꾸는데,

블룸 필터 배열은 char 배열로 한 칸의 크기가 1 byte = 8 bits 이므로

배열의 한 칸을 8칸으로 쪼개서 사용하고 있음을 알 수 있다.

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426561-9998ffc6-d182-497e-8cc2-8a92f6a5611e.png)


데이터를 Read할때 사용하는 KeyMayMatch 함수의 경우

CreateFilter와 거의 동일한 연산을 사용하나

or  대신 and 연산을 사용하여 필터 내부에 특정 key 값이 존재하는지 확인하는 것을 볼 수 있다.

 

 

 

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426584-f700c3df-935e-406d-a918-540a42c7635b.png)

 


다시 코드 흐름을 살펴보자면,

class Benchmark에서 클래스 변수 filterpolicy를 선언하고

생성자 Benchmark()에서 해당 변에 Bloomfilterpolicy 값을 채웠다.

이제 이를 바탕으로 Run() 클래스 함수를 실행하게 된다.

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426611-de6ca39a-c5a3-4023-a8a1-eb495d5cb9b5.png)



Run() 클래스 함수 역시 크게 3가지 부분으로 나뉘는데

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426642-fe0298bb-797c-4453-8baf-877f042e7468.png)


PrintHeader()는 db_bench를 돌리는 환경이나 key-vaule 설정을 출력해주는 함수이다.

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426672-0d944475-79c4-432c-ba36-14df703c97f6.png)


그 다음 Open 함수의 경우

Options이란 클래스를 새롭게 생성하여

filterpolicy를 포함하여 캐시나 버퍼 사이즈 등 변수를 저장하고

이를 인자로 DB::Open 함수를 실행한다.


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426707-58f23dad-5837-46e5-aa3d-a652ff239d58.png)
 

이 다음 DB::Open의 경우도 인자로 받은 options의 값들을 impl이란 변수에 넣고

이를 인자로 MaybeScheduleCompaction() 함수를 수행한다.

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426744-152b6145-f791-45f1-b45e-3e8c5139930f.png)


디 다음 흐름은 위와 같은데 Compaction을 진행하고

필터 블록을 포함하여 SSTable을 생성한다.

이 과정에선 filter_policy는 이전 함수에서 다음 함수로 값을 전달 받기만 하므로 자세한 설명은 생략한다.


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426775-4a824072-59c7-41dc-bb55-70ff53007dc4.png)


*참고로 블룸 필터는 SSTable에 생성되므로,

전체 데이터 크기가 적어 compaction이 발생하지 않는다면

BGWork()가 아닌 NeedsCompaction()이 수행되어 블룸 필터가 생성되지 않는다.

 

 ![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426808-21842d9f-e759-45ce-8a3c-1228726f1928.png)



이후 마지막 GenerateFilter 함수에서,

지금까지 유지해온 filterpolicy의 createFilter 함수를 호출하여 write를 진행한다.
