Git .gitignore 模式匹配规则
1. 基本规则
相对于仓库根目录：所有的模式都是相对于 .gitignore 文件所在的目录（通常是仓库根目录）进行匹配的。
2. 文件和文件夹匹配
```gitignore
# 匹配任何位置的 .idea 文件夹
.idea/

# 匹配任何位置的 config.json 文件
config.json

# 只匹配根目录下的 temp 文件夹
/temp/

# 匹配任何位置的 node_modules 文件夹
node_modules/
```
3. 路径匹配规则
递归匹配（默认行为）
```gitignore
# 这些会匹配项目中任何位置的对应文件/文件夹：
*.log                    # 任何地方的 .log 文件
.idea/                   # 任何地方的 .idea 文件夹
node_modules/            # 任何地方的 node_modules 文件夹
```
实际匹配的路径示例：
project/.idea/
project/subdir/.idea/
project/a/b/c/.idea/
project/family_ultimatum_cloud/.idea/
精确路径匹配
```gitignore
# 以 / 开头表示从根目录开始的精确路径
/build/                  # 只匹配根目录的 build 文件夹
/src/temp.txt           # 只匹配根目录 src 下的 temp.txt

# 包含 / 的路径（但不以 / 开头）
docs/build/             # 匹配任何位置的 docs/build/ 路径
```
4. 通配符使用
```gitignore
# 星号 * 匹配任意字符（除了 /）
*.txt                   # 所有 .txt 文件
temp*                   # 以 temp 开头的文件/文件夹

# 双星号 ** 匹配任意层级的目录
**/logs/                # 任何层级的 logs 文件夹
logs/**                 # logs 文件夹下的所有内容
**/logs/*.log           # 任何层级 logs 文件夹下的 .log 文件

# 问号 ? 匹配单个字符
test?.txt               # test1.txt, testA.txt 等
```
5. 实际例子解析
基于你的项目结构，让我举几个具体例子：
```gitignore
# 你当前的配置
.idea/                  # ✅ 匹配所有位置的 .idea 文件夹
                       #    包括 family_ultimatum_cloud/.idea/

# 其他常见配置
target/                 # ✅ 匹配 family_ultimatum_cloud/target/
*.class                 # ✅ 匹配任何地方的 .class 文件
node_modules/          # ✅ 匹配任何地方的 node_modules

# 精确匹配示例
/family_ultimatum_cloud/target/    # ❌ 这样写是错误的，太具体了
family_ultimatum_cloud/target/     # ✅ 只匹配这个特定路径

# 特定子目录匹配
wx_mini/node_modules/  # ✅ 只匹配 wx_mini 下的 node_modules

```
6. 文件夹 vs 文件的区别
```gitignore
# 只匹配文件夹
temp/                   # 只匹配名为 temp 的文件夹

# 匹配文件和文件夹
temp                    # 匹配名为 temp 的文件或文件夹

# 只匹配文件
temp.txt               # 只匹配文件（因为有扩展名）
```
7. 优先级和覆盖
```gitignore
# 先忽略所有 .txt 文件
*.txt

# 但不忽略 important.txt
!important.txt

# 忽略整个 build 目录
build/

# 但不忽略 build 目录下的 .gitkeep 文件
!build/.gitkeep