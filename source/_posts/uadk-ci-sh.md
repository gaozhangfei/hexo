---
title: uadk_ci.sh
date: 2022-11-23 12:03:50
tags: [tools, ci]
description: use uadk ci builder.sh build and test pull request
---

We can use uadk ci builder.sh to build uadk and openssl-uadk on both ARM and X86.
ARM: build and sanity_test
X86: only build

Moreover, we can use builder.sh to test specific pull request.
For example, testing https://github.com/Linaro/uadk/pull/529
Need manually change uadk/builders.sh.

```
$ git clone https://git.linaro.org/ci/job/configs
$ cd configs/uadk

diff --git a/uadk/builders.sh b/uadk/builders.sh
index 28f55178b..2ca49cfd8 100755
--- a/uadk/builders.sh
+++ b/uadk/builders.sh
@@ -105,6 +105,7 @@ check_existed_repo() {
git clone --depth 1 https://github.com/Linaro/uadk.git
fi
		cd ${WORKSPACE}/uadk
+               git pull origin pull/529/head
		;;
	"ssluadk")
		if [ ! -d ${WORKSPACE}/uadk_engine ]; then

$ ./builder.sh
```
