---
title: 敏感词搜索-DFA算法实现(附Php和Node版)
date: 2019-01-06 13:56:59
tags: dfa
---

## 背景
最近项目需要做敏感词过滤的功能，调研后发现，有以下个方案可以选择：
1. 直接将敏感词写进数组，利用查询数组的方法来查询。
2. 传统的敏感词入库后SQL查询。
3. 利用ES搜索引擎建立分词索引来查询。
4. 利用DFA算法来进行。

首先，产品经理收集到的敏感词有三千多条，且未来还会增加，使用第一种方案肯定不行。  
其次，为了方便以后的扩展性尽量减少对数据库的依赖，以及对敏感词检测接口的性能考虑，放弃第二个方案。  
第三个方案，使用ES成本太大且依赖其他部门，敏感词增加后需要触发更新索引，太麻烦，放弃。  
于是我们选定4方案为研究目标。

## DFA简介
直接移步维基百科->[Deterministic_finite_automaton](https://en.wikipedia.org/wiki/Deterministic_finite_automaton)  
>简单来说它是是通过event和当前的state得到下一个state，即event+state=nextstate。理解为系统中有多个节点，通过传递进入的event，来确定走哪个路由至另一个节点，而节点是有限的。

## 举个栗子

假设我们现在有以下敏感词：
+ 保安
+ 保姆
+ 搬运工

那我们可以将它们构造成二叉树:
![](/images/201901/dfa.png)

使用json格式可表示为：
```json
{
  "保": {
    "安": {
      "end": 1
    },
    "姆": {
      "end": 1
     }
  },
  "搬": {
    "运" : {
     "工": {
      "end": 1
     }
    }
  }
}
```

这种格式下，查询性能很好，如果文本字符串长度为n，模式字符串长度为m，则时间复杂度为O(m+n)

## 实现

### PHP版本

**第一步，我们需要把原始敏感词转换为上诉二叉树格式(需要用到递归)**
```php
<?php
/**
 * Author:LTS
 * Datetime: 2018/9/18 10:12
 */
namespace lib;

class InitWordsMap
{
    /**
     * 加载敏感词库生成树形结构存进文件
     */
    public function loadSensitiveWords()
    {
        $sensitive_words = [];
        $file = fopen(__DIR__ . '/../sensitive-words-raw2.txt', 'r');
        if ($file) {
            while (!feof($file)) {
                $line = trim(fgets($file));
                $this->mergePathToMap($sensitive_words, $line);
            }
        }
        file_put_contents(__DIR__ . '/../sensitive-words2.txt', json_encode($sensitive_words));
    }

    /**
     * 将字符串变成树形结构
     * @param $str
     * @param int $length
     * @return array
     */
    private function getStrPath($str, $length = 0)
    {
        $result = [];
        $key = mb_substr($str, $length, 1);
        if ($length === mb_strlen($str) - 1) {
            $result[$key] = [
                'end' => 1
            ];
            return $result;
        } else {
            $result[$key] = $this->getStrPath($str, $length + 1);
            return $result;
        }
    }

    /**
     * 将字符串变成树形结构并插入进敏感词map
     * @param $sensitive_words_map
     * @param $str
     * @param int $length
     */
    private function mergePathToMap(&$sensitive_words_map, $str, $length = 0)
    {
        $key = mb_substr($str, $length, 1);
        if ($length === mb_strlen($str) - 1) {
            $sensitive_words_map[$key] = [
                'end' => 1,
            ];
        } else {
            if ($sensitive_words_map[$key]) {
                $this->mergePathToMap($sensitive_words_map[$key], $str, $length + 1);
            } else {
                $sensitive_words_map[$key] = $this->getStrPath($str, $length + 1);
            }
        }
    }
}
```

**第二步，生成map数据后，进行查询**
支持递归查询和迭代查询,递归查询比较耗内存，优先使用迭代。

php实现：
```php
    /**
     * 迭代版检查dfa关键词，推荐使用
     * @param $q
     * @return bool
     */
    public function iterateCheck($q)
    {
        $nowMap = $this->sensitive_words_map;
        for ($i = 0; $i < mb_strlen($q); $i++) {
            $key = mb_substr($q, $i, 1);
            if (in_array($key, $this->excludes)) {
                continue;
            }
            if ($nowMap[$key]) {
                if ($nowMap[$key]['end'] === 1) {
                    return true;
                } else {
                    $nowMap = $nowMap[$key];
                }
            } else {
                $nowMap = $this->sensitive_words_map;
            }
        }
        return false;
    }

    /**
     * 递归版dfa查询，数据量少时推荐，数据量大占内存
     * @param array $sensitive_words_map
     * @param string $q
     * @param int $length
     * @return bool
     */
    public function recurCheck($q = '', $sensitive_words_map = [], $length = 0)
    {
        if (!$sensitive_words_map) {
            $sensitive_words_map = $this->sensitive_words_map;
        }
        $key = mb_substr($q, $length, 1);
        if (in_array($key, $this->excludes)) {
            return $this->recurCheck($q, $sensitive_words_map, $length + 1);
        }

        if ($sensitive_words_map[$key]) {
            if ($sensitive_words_map[$key]['end'] === 1) {
                return true;
            } else {
                return $this->recurCheck($q, $sensitive_words_map[$key], $length + 1);
            }
        } else {
            if (mb_strlen($q) === 1) {
                return false;
            }
            return $this->recurCheck(mb_substr($q, $length + 1), $this->sensitive_words_map);
        }
    }
```

### node版本
node实现和php类似，完整代码如下(执行node index.js即可运行)：

```js
const http = require('http')
const url = require('url')
const readline = require('readline')
let fs = require('fs');

(async () => {
  let excludes = ['@', 'x', '$', '&', '\\', '/', '|']
  let sensitiveWordsMap = {}
  function getQ(query, body)
  {
    let q = query.q
    if (body) {
      try {
        let bodyJson = JSON.parse(body)
        if (bodyJson.q) {
          q = bodyJson.q
        }
      } catch (e) {
      }
    }
    return q
  }

  async function loadSensitiveWords()
  {
    let fRead = fs.createReadStream('./words.txt')
    let rl = readline.createInterface({input: fRead});
    await new Promise(resolve => {
      rl.on('line', line => {
        mergePathToMap(sensitiveWordsMap, line)
      }).on('close', () => {
        resolve()
      })
    })
  }

  function getStrPath(str, length = 0) {
    let result = {}
    let key = str.charAt(length)
    if (length === str.length - 1) {
      result[key] = {
        'end':1
      }
      return result
    } else {
      result[key] = getStrPath(str, length + 1)
      return result
    }
  }

  function mergePathToMap(sensitiveWordsMap, str, length = 0)
  {
    let key = str.charAt(length)
    if (length === str.length - 1) {
      sensitiveWordsMap[key] = {
        'end':1
      }
    } else {
      if (typeof sensitiveWordsMap[key] !== 'undefined') {
        mergePathToMap(sensitiveWordsMap[key], str, length + 1)
      } else {
        sensitiveWordsMap[key] = getStrPath(str, length + 1)
      }
    }
  }

  /**
   * 递归版检查，q数据量少时推荐，占内存
   * @param nowMap
   * @param q
   * @param length
   * @returns {*}
   */
  function recurCheck(q, nowMap, length = 0) {
    if (!nowMap) {
      nowMap = sensitiveWordsMap
    }
    let key = q.charAt(length)
    if (excludes.indexOf(key) !== -1) {
      return recurCheck(q, nowMap, length + 1)
    }
    if (typeof nowMap[key] !== 'undefined') {
      if (nowMap[key]['end'] === 1) {
        return true
      } else {
        return recurCheck(q, nowMap[key], length + 1)
      }
    } else {
      if (q.length === 1) {
        return false
      }
      return recurCheck(q.substr(length + 1), sensitiveWordsMap)
    }
  }

  /**
   * 递归检查关键词，推荐使用
   * @param q
   * @returns {boolean}
   */
  function iterateCheck (q) {
    let nowMap = sensitiveWordsMap
    for (let i = 0; i < q.length; i++) {
      let key = q.charAt(i)
      if (excludes.indexOf(key) !== -1) {continue}
      if (typeof nowMap[key] !== 'undefined') {
        if (nowMap[key]['end'] === 1) {
          return true
        } else {
          nowMap = nowMap[key]
        }
      } else {
        nowMap = sensitiveWordsMap
      }
    }
    return false
  }

  await loadSensitiveWords()
  http.createServer(function (request, response) {
    let body = '' // 请求正文
    request.setEncoding('utf-8')
    request.on('data', function (chunk) {
      body += chunk
    }).on('end', function () {
      (async () => {
        response.writeHead(200, {
          'Content-Type': 'application/json',
        })
        const query = url.parse(request.url, true).query // 查询参数
        let q = getQ(query, body)

        if (!q) {
          response.write(JSON.stringify({
            'code': -1,
            'msg': 'need q',
          }))
          response.end()
          return
        }

        response.write(JSON.stringify({
          'code': 0,
          'isExists': recurCheck(q)
        }))
        response.end()
      })().catch((error) => {
        console.log(error)
        response.writeHead(500, {
          'Content-Type': 'text/plain',
        })
        response.write('Internal Server Error')
        response.end()
      })
    })
  }).listen(8081)
})()
```

## 后记
. 推荐使用node版本，因为php每次访问都会初始化map，会损耗一定的时间，而node只需要在启动的时候初始化map即可。
