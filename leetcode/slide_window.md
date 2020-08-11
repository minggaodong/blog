# 滑动窗口解析
## 概述
滑动窗口用于解决两个字符串间比较的问题，比如在一个字符串中寻找符合另一个字符串某种特征的子串。

滑动窗口一般需要用到“双指针”来进行求解，右指针右移入窗，左指针右移出窗。

验证字符时，需要借助于 unordered_map 来实现 O(1) 查询，使用 count() 方法判断 key 是否存在；使用中括号 map[key] 访问不存在的 key 时，c++ 默认会创建这个 key，并赋值为 0 。

### 解题框架
```
int left = 0, right = 0;

while (right < s.size()) {
    window.add(s[right]);
    right++;
    
    while (valid) {
        window.remove(s[left]);
        left++;
    }
}
```
- 滑动窗口的解题框架就是定义两个左右指针
- 右指针右移获取字符，加入窗口
- 判断新字符是否符合要求，如果不符合需要收缩窗口
- 将左指针右移来收缩窗口，收缩到符合要求的位置

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

### 567. 字符串的排列
给定两个字符串 s1 和 s2，写一个函数来判断 s2 是否包含 s1 的排列。

换句话说，第一个字符串的排列之一是第二个字符串的子串。

#### 示例1:
```
输入: s1 = "ab" s2 = "eidbaooo"
输出: True
解释: s2 包含 s1 的排列之一 ("ba").
```

#### 示例2:
```
输入: s1= "ab" s2 = "eidboaoo"
输出: False
```

#### 思路
- 创建一个 unordered_map s1 中字符出现的次数。
- 右指针移动一位，获取一个新字符，设置unordered_map的值减1，表示入窗。
- 判断如果新字符在 map 中 value 为 -1，则说明新字符是非法的，需要收缩窗口、
- 移动左指针收缩窗口，设置unordered_map的值加1，代表出窗；一直收缩到新字符值不为-1为止

#### 代码
```
bool checkInclusion(string s1, string s2) {
    unordered_map<char, int> mp;
    for (auto &c: s1) mp[c]++; // 记录 出现次数的差值

    int l = 0, r = 0;
    while (r < s2.size()){
        char c = s2[r++];
        mp[c]--; // 入窗
        while (l < r && mp[c] < 0){ // 出窗
            mp[s2[l++]] ++;
        }
        if (r - l == s1.size()) return true;
    }
    return false;
}
```
### 209. 长度最小的子数组
给定一个含有 n 个正整数的数组和一个正整数 s ，找出该数组中满足其和 ≥ s 的长度最小的 连续 子数组，并返回其长度。如果不存在符合条件的子数组，返回 0。
#### 示例：
```
输入：s = 7, nums = [2,3,1,2,4,3]
输出：2
解释：子数组 [4,3] 是该条件下的长度最小的子数组。
```

#### 代码
```
   int minSubArrayLen(int s, vector<int>& nums) {
        int min_size = 0;
        int win_sum = 0;
        
        int l = 0, r = 0;
        while (r < nums.size()) {
            win_sum += nums[r++];	 // 入窗
            //printf("in win: num=%d, win_sum=%d\n", nums[r-1], win_sum);
            while (l < r && win_sum > s) {
                if (win_sum - nums[l] < s)
                    break;
                win_sum -= nums[l++]; // 出窗
                //printf("out win: num=%d, win_sum=%d\n", nums[l-1], win_sum);
            }

            if (win_sum >= s && ((min_size == 0 || r - l < min_size)))
                min_size = r - l;
            //printf("min_size=%d, curr_size=%d\n", min_size, r-l);
        }
        
        return min_size;
    }
```
