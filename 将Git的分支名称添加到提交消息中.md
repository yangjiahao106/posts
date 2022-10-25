---
title: 将Git的分支名称添加到提交消息中
date: 2022-07-27 01:07:43
tags:
- git
---

## 实现方法

- 在 .git/hooks 目录下创建 prepare-commit-msg文件，并将下面的脚本放入文件中，

```bash
#!/bin/bash
  
# This way you can customize which branches should be skipped when
# prepending commit message. 
if [ -z "$BRANCHES_TO_SKIP" ]; then
  BRANCHES_TO_SKIP=(master develop test)
fi

BRANCH_NAME=$(git symbolic-ref --short HEAD)
#BRANCH_NAME="${BRANCH_NAME##*/}"

BRANCH_EXCLUDED=$(printf "%s\n" "${BRANCHES_TO_SKIP[@]}" | grep -c "^$BRANCH_NAME$")
BRANCH_IN_COMMIT=$(grep -c "\[$BRANCH_NAME\]" $1)

if [ -n "$BRANCH_NAME" ] && ! [[ $BRANCH_EXCLUDED -eq 1 ]] && ! [[ $BRANCH_IN_COMMIT -ge 1 ]]; then
  sed -i.bak -e "1s|^|[$BRANCH_NAME] |" $1
fi
```

- 为文件添加可执行权限

``` bash
chmod 755 prepare-commit-msg
```

完成之后分支名就会出现在提交信息里了, 例如：

    Date:   Mon Oct 10 15:00:47 2022 +0800

        [feature/cms]  fix bug

