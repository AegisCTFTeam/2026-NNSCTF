- solved by @Yanamachi
- 하단의 과정과 익스플로잇 코드를 이용해 플래그를 획득할 수 있습니다.
![image](../../image/ASS-1.png)
![image](../../image/ASS-2.png)
![image](../../image/ASS-3.png)
![image](../../image/ASS-4.png)
```
import base64

from cryptography.hazmat.primitives import serialization

PRIVATE_KEY = """-----BEGIN PRIVATE KEY-----
MC4CAQAwBQYDK2VwBCIEILBwlah+2FwhK+0uwLAM8QwUorlX1GvO64pwovlRVsE9
-----END PRIVATE KEY-----
"""

NONCE = "da39e5b8c6dea33978707ae7bd53988209f61d0f668702348b28cc70485ca0fa"

key = serialization.load_pem_private_key(
    PRIVATE_KEY.encode(),
    password=None,
)

signature = key.sign(NONCE.encode())

print(base64.b64encode(signature).decode())
```
![image](../../image/ASS-5.png)
- FLAG: NNS{Rfc_3454_fRoze_7He_t4b1e_bU7_tHe_uniC0D3_KePt_walk1ng_Cr4ZY_ri6ht}
