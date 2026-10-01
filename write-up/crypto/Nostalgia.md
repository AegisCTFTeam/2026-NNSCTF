- solved by @cooku222
  
lcg(s) 함수 내부에서 int(s)를 호출한다. 즉 seed값으로 time.time()을 넣어도, 첫 번째 LCG 호출에서 소수점 이하가 버려지고 정수 Unix 타임스탬프만 남게 된다. 시드 값이 한정적이라 브루트포스가 현실적인 시간 내에서 가능하게 된다. 

참고로 계산할 때 하단과 같은 방법으로 시간을 단축할 수 있다.
LCG는 s → a·s + b (mod 2^32) 형태의 선형(affine) 함수이다.
1337번 반복을 미리 합성해서 상수 두 개로 압축했다. 

A = 586823937, B = 3067822255
최종_seed = (A * timestamp + B) mod 2^32


최종 seed를 계산했으면, sha256(str(seed))로 AES 키를 생성해준다. 이 코드에서의 AES 키와 동일하고, 코드에 나온대로 ECB 복호화를 해준다.

FLAG: NNS{th3_b3st_t1m3_t0_m4k3_m3m0r13s_15_n0w}
