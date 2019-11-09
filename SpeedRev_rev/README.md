# Speed Rev

#### 설명

- 간단한 C 프로그램
- 암호화된 문자열을 복구하는 것이 목표이다.

```c
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  unsigned int v3; // eax
  unsigned __int64 i; // [rsp+8h] [rbp-8h]

  v3 = time(0LL);
  srand(v3);
  sub_AB1();
  for ( i = 0LL; i <= 26; ++i )
    putchar(byte_202010[i]);
  putchar(10);
  return 0LL;
}
```

그냥 단순하게 sub_AB1() 함수를 호출한 후, 전역변수를 26바이트 출력하는게 전부이다.

하지만 byte_202010로 가면 알 수 없는 문자열이 있기 때문에 sub_AB1() 함수에서 byte_202010를 변형했다는것을 알 수 있다.

sub_AB1() 함수 내부로 들어가다 보면 byte_202010를 변형하는 다음과 같은 함수를 찾을 수 있다.

```c
void __fastcall sub_A57(__int64 a1)
{
  unsigned __int64 i; // [rsp+18h] [rbp-8h]

  for ( i = 0LL; i <= 8; ++i )
    byte_202010[a1] ^= sub_83A(a1 & 0xF);
}
```

sub_A57함수는 호출하는측에서 a1 0~26까지 차례대로 넣고 호출된다.

여기서 첫번째 함정이 있는데, i는 반복문말고는 전혀 사용하지 않는다는점과 i의 조건이 `i <= 8` 이기 때문에 8번이 아니라 9번 반복문이 돈다는 점이다.

그렇기 때문에 이 함수는

```c
byte_202010[a1] ^= sub_83A(a1 & 0xF);
```

와 동일하다.

다시 sub_83A에 들어가보면 다음과 같은 코드를 볼 수 있다.

```c
__int64 __fastcall sub_83A(int a1)
{
  unsigned __int64 i; // [rsp+18h] [rbp-28h]
  char *v3; // [rsp+20h] [rbp-20h]
  char salt; // [rsp+2Ch] [rbp-14h]
  char v5; // [rsp+2Dh] [rbp-13h]
  char key[10]; // [rsp+2Eh] [rbp-12h]
  unsigned __int64 v7; // [rsp+38h] [rbp-8h]

  v7 = __readfsqword(0x28u);
  do
  {
    salt = byte_202030[a1] % 0x19u + 97;
    v5 = byte_202030[(a1 + 1) % 16] % 0x19u + 97;
    for ( i = 0LL; i <= 8; ++i )
      key[i] = rand() % 25 + 97;
    LOBYTE(v7) = 0;
    v3 = crypt(key, &salt);
  }
  while ( v3[2] != byte_202030[a1 % 16]
       || v3[3] != byte_202030[(a1 + 1) % 16]
       || v3[4] != byte_202030[(a1 + 2) % 16]
       || v3[5] != byte_202030[(a1 + 3) % 16] );
  return (unsigned __int8)v3[2];
}
```

약간 복잡해졌는데 이번에 새로운 전역변수 byte_202030이 나왔다.

do-while문 내부 루프에 복잡해보이는 코드가있는데 이것이 2번째 함정이다.

결국 이 함수의 리턴값은 v3[2]이고, do-while문을 탈출하려면 꼭 v3[2]는 byte_202030[a1 % 16] 이여야한다.

그래서 이 함수는

```c
int sub_83A(int a1){
    return byte_202030[a1 % 16]
}
```

와 동일하다.

여기까지 나온 사실을 종합하면 그냥 byte_202010를 byte_202030에 있는 16글자 문자열을 키로 xor해주면 된다는 사실을 알 수 있다.

다음처럼 간단하게 짜면 플래그를 얻을 수 있다.

```python
a = [
    0x12, 0x24, 0x08, 0x14, 0x3B, 0x22, 0x1A, 0x47, 0x31, 0x69, 
    0x1B, 0x49, 0x08, 0x41, 0x20, 0x0F, 0x2B, 0x17, 0x0A, 0x12, 
    0x34, 0x36, 0x00, 0x45, 0x00, 0x57, 0x09
]
b = [
    0x74, 0x48, 0x69, 0x73, 0x40, 0x69, 0x73, 0x2A, 0x6E, 0x30, 
    0x74, 0x26, 0x66, 0x6C, 0x41, 0x67
]

flag = ''
for n, x in enumerate(a):
    flag += chr(x ^ b[n % 16])
print(flag) # flag{Kim_Yoon-ah__cat_song}
```

좋은 노래이다 들어보자
