---
title: vim command
feed: show
date: 23-10-2023
permalink: /vim-command
---
따로 정리할만한 재미있는 vim 커맨드 모음입니다.

- `viw` : **v**isual mode "**i**nner **w**ord"
```
					v_iw iw
"inner word", select [count] words (see word).
White space between words is counted too.
When used in Visual linewise mode "iw" switches to
Visual characterwise mode.
```

visual mode로 들어가서 단어 하나를 선택합니다.

마찬가지로 `vi(`, `vi'` 와 같이 괄호나 따옴표 안의 컨텐츠를 선택할 수 있습니다.

선택한 후에는 주로 `c` 로 지우고 바로 대안을 작성하거나 혹은 `y`, `d` 를 사용합니다.