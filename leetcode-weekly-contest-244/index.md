# LeetCode 第244场周赛题解


### Problem A - 判断矩阵经轮转后是否一致
**题解**：
- 算法：暴力枚举
- 思路：将$mat$这个矩阵分别顺时针旋转 90度、180度、270度、360度，然后判断是否和 $target$ 矩阵相等，只要存在一个旋转方案相等，则返回 true, 否则返回 false。 所以我们只需要编写两个函数即可，一个函数用来旋转 $mat$ 矩阵，一个函数用来判断两个矩阵是否相同
- 矩阵旋转函数：我们只需要枚举矩阵的左上部分旋转90、180、270、360度即可，注意矩阵长度是奇偶的情况（画图即可找到规律）。其中一个点最多能旋转出四个不同的点，用上下左右表示，可以发现四个点中上下对称，左右对称。我们知道了上面的点坐标$(x,y)$就容易写出下面的点的坐标$(n-1-x, n-1-y)$，同理于左右两点。假设第一个点是$(x,y)$, 关键得到第二个点就可以推出其他的两个点了，第二个点可以简单画图得出规律 $(y, n - 1 - x)$。
- 矩阵比较函数：这个函数其实可以不用写，vector重载了 operator== ，所以可以直接用 $mat==target$来判断矩阵是否相同。当然也可以写一个两重循环进行判断
```C++
int n, m;
class Solution {
public:
    void rotate(vector<vector<int>>& mat){
        for(int i = 0; i < n / 2; ++i){
            for(int j = 0; j < (m + 1) / 2; ++j){
                int &v = mat[i][j];
                int x = j, y = n - 1 - i;
                swap(mat[x][y], v);
                x = n - 1 - i, y = n - 1 - j;
                swap(mat[x][y], v);
                x = n - 1 - j, y = i;
                swap(mat[x][y], v);
            }
        }
    }
    bool check(vector<vector<int>>& mat, vector<vector<int>>& target){
        for(int i = 0; i < n; ++i){
            for(int j = 0; j < m; ++j){
                if(mat[i][j] != target[i][j]) return false;
            }
        }
        return true;
    }
    bool findRotation(vector<vector<int>>& mat, vector<vector<int>>& target) {
        n = mat.size(), m = mat[0].size();
        for(int i = 0; i <= 4; ++i){
            // 此处可以直接写成 if(mat == target) return true;
            if(check(mat, target)) return true;
            rotate(mat);
        }
        return false;
    }
};
```
空间复杂度：$O(N^2)$ \
时间复杂度：$O(N^2)$

### Problem B - 使数组相等的减少操作次数
**题解**：
- 算法：枚举遍历、排序
- 思路：将数组从大到小排序，数组可以被看成是由一段一段连续相等的数字组成，从大到小统计数字；使用双指针可以计算出从$i$开始相等的数字的结尾部分$j$，对于当前这段数字来说，它的个数应该是$j$,因为之前的所有比它大的数字都已经通过减小操作变成了和当前这段数字相等的数。
```C++
class Solution {
public:
    int reductionOperations(vector<int>& nums) {
        int n = nums.size();
        sort(nums.rbegin(), nums.rend());
        int ans = 0;
        for(int i = 0; i < n; ++i){
            int j = i;
            while(j < n && nums[i] == nums[j]) j += 1;
            if(j >= n) break;
            ans += j;
            i = j - 1;
        }
        return ans;
    }
};
```
空间复杂度：$O(N)$\
时间复杂度：$O(N)$

### Problem C - 使二进制字符串交替的最小反转次数
**题解**：
- 思路：字符串不管怎么操作，最终只有两种可能情况，’101010...‘ 形式的字符串和 '010101...'形式的字符串。因此我们分别计算当前字符串转换成这两种字符串所需的代价即可。
- 计算代价函数：因为字符串s有两种操作，将开头元素append到最后，或者将反转一个字符。我们的目标是求出最小的反转代价。
第一种操作的本质是将字符串的前一段子串移动到字符串的尾部，这里使用一个常用的技巧来考虑所有的append情况，将字符串s复制一份放在字符串s的尾部，记为$S$，那么我们用一个长度为s.size()的窗口不断滑动，就等价于枚举了所有的append情况；至此，问题转换成了，如果知道了一个字符串$s^'$变成字符串$t$的最小反转代价，当将字符串$s^'$的首部元素移动到尾部,记为$s^{''}$，如何计算出$s^{''}$变成字符串$t$的最小反转代价？观察可得，当我们统计了字符串$s^'$和字符串$t$的所有对应位置相同的元素和不同的元素个数之后，当我们将$s^{'}$变换成$s^{''}$，所有本来$s^{'}$和$t$相同的位置的元素，现在都变成了不同的，而本来不同的元素，现在变成了相同（要特别处理一下头和尾元素，细节看代码）。 其中，字符串$s^'$变成字符串$t$的最小反转代价就很简单了，就是它们在相同位置下不同元素的个数。
```C++
class Solution {
    int ans = INT_MAX;
    int n;
public:
    int minFlips(string s) {
        string res = s + s;
        n = s.size();
        solve(res, '1');
        solve(res, '0');
        return ans;
    }
    void solve(string &res, char t){
        int a = 0, b = 0;
        int m = res.size();
        char tmp = t;
        // 先得出第一个窗口所需的代价
        for(int i = 0; i < n; ++i){
            if(res[i] == t) a++;
            else b++;
            t ^= 1;
        }
        t ^= 1;
        ans = min(ans, b);
        // 移动窗口
        for(int l = 0, r = n; r < m; ++r){
            swap(a, b);
            // 处理尾部
            if(res[r] == t) a += 1;
            else b += 1;
            // 处理头部
            if(res[l] == tmp) b -= 1;
            else a -= 1;
            l += 1;
            ans = min(ans, b);
        }
    }
};
```
空间复杂度：$O(|S|)$\
时间复杂度：$O(|S|)$

### Problem D - 装包裹的最小浪费时间
**题解：**
- 思路：对于困难的问题，我的常用做法是先将问题弱化，看看能不能找到解决方案，如果能的话，再看看加强之后应该做些什么变换。这个问题的弱化版本就是，我们只有一个供应商，那么问题就非常好做了，我们枚举每个物品，找到最小的大于它的箱子（这一步显然二分即可），然后累计代价即可。那么对于这个问题来说，有多个供应商，我们不能直接对箱子进枚举，转换下思路，我们是否可以每个供应商进行枚举，枚举它可以装的最大的物品。

```C++
const int MOD = 1000000007;
typedef long long LL;
class Solution {
public:
    int minWastedSpace(vector<int>& p, vector<vector<int>>& b) {
        sort(p.begin(), p.end());
        
        int n = p.size();
        vector<LL>sum(n+1);
        for(int i = 0; i < n; ++i){
            sum[i] = p[i];
            if(i) sum[i] += sum[i-1];
        }
        LL ans = 1e12;
        for(auto &v : b){
            int m = v.size();
            sort(v.begin(), v.end());
            if(v.back() < p.back()) continue;
            int l = 0;
            LL res = 0;
            for(int i = 0; i < m; ++i){
                auto iter = upper_bound(p.begin() + l, p.end(), v[i]);
                if(iter == p.begin()) continue;
                int r = iter - p.begin() - 1;
                res += 1ll * (r - l + 1)  * v[i];
                if(l > 0) res = res - sum[r] + sum[l-1];
                else res = res - sum[r];
                l = r + 1;
            }
            ans = min(ans, res);
        }
        if(ans == 1e12) return -1;
        return ans % MOD;
    } 
};
```
空间复杂度：$O(N)$\
时间复杂度：$O(Nlog(N)+Mlog(N))$，其中$N$表示货物的个数，$M$表示供应商的箱子总数

>请各位看官批评指正，接受任何**善意**的建议
