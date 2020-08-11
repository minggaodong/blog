# 滑动窗口解析
## 概述




## 经典题型
### 76. 最小覆盖子串
给你一个字符串 S、一个字符串 T 。请设计一种算法，可以在 O(n) 的时间复杂度内，从字符串 S 里面找出：包含 T 所有字符的最小子串。

#### 示例：
```
输入：S = "ADOBECODEBANC", T = "ABC"
输出："BANC"
```

#### 提示：
```
如果 S 中不存这样的子串，则返回空字符串 ""。
如果 S 中存在这样的子串，我们保证它是唯一的答案。
```

#### 思路
- 采用双指针维护一个滑动窗口，通过窗口扩充和收缩寻找到一个满足要求的字串，并维护当前最小字串的起始位置和长度。
- 使用 unordered_map 保存子串，用来O(1)复杂度验证字符是否属于子串
- 使用 unordered_map 记录窗口内保存的字符，由于字符可能重复，value值为字符出现的次数。
- 右指针向右移动，寻找目标字符，当找到全部目标字符时，停止右移，此时窗口内是存在冗余的。
- 左指针向右移动，收缩窗口消除冗余，只到窗口不包含全部字符为止，每次移动前都更新最小子串的起始位置和长度。

#### 代码
```
string minWindow(string s, string t) {
    int left = 0, right = 0, start = 0, minLen = INT_MAX;
    unordered_map<char, int> need;
    unordered_map<char, int> window;
    for (char c : t) {
        need[c]++; // 目标字符数
    }
    int match = 0;
    while (right < s.size()) { // 滑窗开始
        char c1 = s[right];
        if (need.count(c1)) { // 当前字符为目标字符
            window[c1]++; // 更新窗口内字符数
            if (window[c1] == need[c1]) { // 若改字符的窗口内字符数 = 需要的目标字符数，匹配字符数加一
                match++;
            }
        }
        right++; // 窗口右扩
        while (match == need.size()) { // 当窗口内的所有字符均已匹配完成，开始窗口左侧缩窗
            if (right - left < minLen) { // 更新当前窗口最小值及窗口起始位置
                minLen = right - left;
                start = left;
            }
            char c2 = s[left];
            if (need.count(c2)) { // 若左侧字符为目标字符
                window[c2]--; // 更新窗口内字符统计
                if (window[c2] < need[c2]) { // 窗口变更后不满足目标字符数要求，匹配字符数减一，循环结束，重新开始新一轮右扩
                    match--;
                }
            }
            left++; // 窗口左缩
        }
    }
    return minLen == INT_MAX ? "" : s.substr(start, minLen);
}
```

### 3. 无重复字符的最长子串
给定一个字符串，请你找出其中不含有重复字符的 最长子串 的长度。

#### 示例 1:
```
输入: "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
```

#### 示例 2:
```
输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
示例 3:

输入: "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
```
#### 思路
- 采用双指针维护一个滑动窗口，维护一个最长不重复字串的长度 max_len。
- 采用 unordered_map 保存窗口中存在的字符，提供 O(1)复杂度查询。
- 右指针右移1位，判断字符是否在窗口中存在，如果不存在，插入 map 并更新 max_len,继续右移
- 如果字符存在，说明重复，则将左指针移动到窗口内重复字符的下一个字符。

#### 代码
```
int lengthOfLongestSubstring(string s) {
    int max_len = 0;
    int l(0), r(0);

    unordered_map<char, int> win;
    while (r < s.size()) {
        if (win.count(s[r]) && l <= win[s[r]]) {
            l = win[s[r]] + 1;
        } else {
            if (r-l+1 > max_len)
                max_len = r-l+1;
        }
        win[s[r]] = r;
        r++;
    }

    return max_len;
}
```
