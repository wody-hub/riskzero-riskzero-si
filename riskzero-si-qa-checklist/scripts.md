# qa-checklist 실행 스크립트 모음

SKILL.md의 선택적 단계에서 사용하는 스크립트. 해당 단계에 진입했을 때만 이 파일을 읽는다.

- **Step 2 (PPTX 분석)** → §1
- **Step 6-1 (더미데이터 생성)** → §2

---

## §1. PPTX 분석 스크립트 (Step 2)

```python
# python-pptx 분석 스크립트 (Bash로 실행)
python3 -c "
from pptx import Presentation
from pptx.util import Inches
import json, sys

prs = Presentation(sys.argv[1])
result = {'slides': []}
for i, slide in enumerate(prs.slides):
    slide_data = {'number': i+1, 'shapes': []}
    for shape in slide.shapes:
        if shape.has_table:
            table = shape.table
            rows = []
            for row in table.rows:
                rows.append([cell.text.strip() for cell in row.cells])
            slide_data['shapes'].append({'type': 'table', 'rows': rows})
        elif shape.has_text_frame:
            text = shape.text_frame.text.strip()
            if text:
                slide_data['shapes'].append({'type': 'text', 'content': text})
    result['slides'].append(slide_data)
print(json.dumps(result, ensure_ascii=False, indent=2))
" "$PPTX_PATH"
```

> python-pptx 미설치 시: `pip3 install python-pptx`

---

## §2. 더미데이터 생성 스크립트 (Step 6-1)

**전제:** SKILL.md 6-1-0의 대상 서버 확인 게이트를 통과한 후에만 실행한다.

### §2-1. 로그인 토큰 획득

```bash
# 1. 로그인하여 토큰 획득
# 사용자에게 테스트 계정 정보를 확인한다 (ID/PW)
TOKEN=$(curl -s -X POST {config.server.backend.baseUrl}{config.auth.loginApi} \
  -H 'Content-Type: application/json' \
  -d '{"{config.auth.loginFields.username}":"$ID","{config.auth.loginFields.password}":"$PW"}' \
  | jq -r '.{config.auth.tokenField}')
```

### §2-2. 파일 첨부 더미 생성

```bash
# 최소 크기 테스트 파일 생성
# 1x1 투명 PNG (68 bytes)
printf '\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR\x00\x00\x00\x01\x00\x00\x00\x01\x08\x06\x00\x00\x00\x1f\x15\xc4\x89\x00\x00\x00\nIDATx\x9cc\x00\x01\x00\x00\x05\x00\x01\r\n\xb4\x00\x00\x00\x00IEND\xaeB`\x82' > /tmp/qa-test.png

# 빈 PDF
echo '%PDF-1.0
1 0 obj<</Type/Catalog/Pages 2 0 R>>endobj
2 0 obj<</Type/Pages/Kids[3 0 R]/Count 1>>endobj
3 0 obj<</Type/Page/MediaBox[0 0 3 3]>>endobj
xref
0 4
0000000000 65535 f
0000000009 00000 n
0000000058 00000 n
0000000115 00000 n
trailer<</Size 4/Root 1 0 R>>
startxref
190
%%EOF' > /tmp/qa-test.pdf

# 파일 업로드 → docId 획득
DOC_ID=$(curl -s -X POST {config.server.backend.baseUrl}{config.dummyData.fileUploadApi} \
  -H "{config.auth.tokenHeader}: {config.auth.tokenPrefix}$TOKEN" \
  -F "files=@/tmp/qa-test.png" | jq -r '.docId')
```

### §2-3. 데이터 등록

```bash
# DTO 구조에 맞게 N건 등록
for i in $(seq 1 {config.dummyData.count}); do
  curl -s -X POST {config.server.backend.baseUrl}/api/{endpoint} \
    -H "{config.auth.tokenHeader}: {config.auth.tokenPrefix}$TOKEN" \
    -H 'Content-Type: application/json' \
    -d "{
      \"field1\": \"{prefix}$(printf '%03d' $i) 테스트 데이터 $i\",
      ...
    }"
  sleep 0.2
done
```

> 실제 필드명과 값은 DTO 분석 결과를 기반으로 동적 생성한다.
