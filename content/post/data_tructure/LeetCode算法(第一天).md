---
typora-root-url: ../../../static
title: "LeetCode算法(第一天)"
date: 2020-09-15T18:57:36+08:00
draft: false
tags: ["数据结构与算法"]
categories: ["data_structure"]
---

## 两数之和
题目描述如下：

	//给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。
	//
	// 你可以假设每种输入只会对应一个答案。但是，数组中同一个元素不能使用两遍。
	//
	//
	//
	// 示例:
	//
	// 给定 nums = [2, 7, 11, 15], target = 9
	//
	//因为 nums[0] + nums[1] = 2 + 7 = 9
	//所以返回 [0, 1]
	//
	// Related Topics 数组 哈希表
	// 👍 9044 👎 0

### 解法1
一开始想到暴力解法，遍历每个值，然后根据target-current得到当前遍历的值需要对应的值，然后遍历剩余的元素，查看是否存在需要的值，如果存在就返回当前遍历的值以及找到的需要的值的下标。代码如下：

	//解法1：暴力解法，不好
	class Solution0 {
	    public int[] twoSum(int[] nums, int target) {
	        for (int i = 0; i < nums.length; i++) {
	            int current=nums[i];
	            for(int j=i+1;j<nums.length;j++){
	                int temp=target-current;
	                if(nums[j]==temp){
	                    return new int[]{i,j};
	                }
	            }
	        }
	        return null;
	    }
	}

### 解法2
然后感觉可以优化，想到使用hash表，就是先将数组中的元素存到hash表中，表的value是元素对应的索引，表的key是元素的值。这样就不会用到内循环了，查找target-current的值，直接从hash表中查找。代码如下：

	//解法2：哈希表
	class Solution1 {
	    public int[] twoSum(int[] nums, int target) {
	        //将数据存放在hash表中,key:数值，val:数据在数组中的索引
	        HashMap<Integer,Integer> hash=new HashMap<>();
	        for (int i = 0; i < nums.length; i++) {
	            hash.put(nums[i],i);
	        }
	        //查找hash表
	        for (int i = 0; i < nums.length; i++) {
	            Integer index=hash.get(target-nums[i]);
	            if(index!=null&&index!=i){
	                return new int[]{i,index};
	            }
	        }
	        return null;
	    }
	}

### 解法3
对于解法2其实可以优化，没必要首先将数组的元素全部放入hash表中，因为已知两数相加的和，以及其中一个数，可以得到另一个数，所以可以一边遍历一边存放。代码如下：

	//解法3：一遍哈希表,一边查找哈希表一边添加元素
	class Solution {
	    public int[] twoSum(int[] nums, int target) {
	        //将数据存放在hash表中,key:数值，val:数据在数组中的索引
	        HashMap<Integer,Integer> hash=new HashMap<>();
	        for (int i = 0; i < nums.length; i++) {
	            int rest=target-nums[i];
	            Integer index=hash.get(rest);
	            if(index!=null){
	                return new int[]{index,i};
	            }
	            hash.put(nums[i],i);
	        }
	        return null;
	    }
	}

## 两数相加
题目描述如下：

	//给出两个 非空 的链表用来表示两个非负的整数。其中，它们各自的位数是按照 逆序 的方式存储的，并且它们的每个节点只能存储 一位 数字。
	//
	// 如果，我们将这两个数相加起来，则会返回一个新的链表来表示它们的和。
	//
	// 您可以假设除了数字 0 之外，这两个数都不会以 0 开头。
	//
	// 示例：
	//
	// 输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
	//输出：7 -> 0 -> 8
	//原因：342 + 465 = 807
	//
	// Related Topics 链表 数学
	// 👍 4889 👎 0

### 解法
一开始看错了题目，以为要得到反转后的链表，所以写了一个反转链表的辅助函数（对于这个题目没有用），代码如下：

    /**
     * 返回一个反转后的链表
     */
    private ListNode reverseListNode(ListNode target){
        if(null==target){
            return null;
        }
        ListNode temp=target;
        target=target.next;
        temp.next=null;
        while(target!=null){
            ListNode next=target.next;
            target.next=temp;
            temp=target;
            target=next;
        }
        return temp;
    }

然后看清了题目，得到这样的思路：同时从两个链表的最左端开始相加，然后相加（包括进位）结果超过了9就产生进位1，如果没有就将进位变为0。需要考虑： **如果到了最右边，也就是最后一位（我是将结果存在了第一个链表中），如果存在进位，但是第一个链表的下一个节点为null，或者没有进位，同时第一个链表的下一个节点为null并且第二个链表的下一个节点不为null，就要为第一个链表的下一个节点分配一个空白节点。** 。代码如下：

    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        if(l1==null){
            return l2;
        }else if(l2==null){
            return l1;
        }
        ListNode listOne = l1;
        ListNode listTwo = l2;
        ListNode res=listOne;
        //相加,如果当前相加位都为null则退出
        int next = 0;
        while (listOne!=null || listTwo!=null) {
            //为了节省空间，将结果放在listOne中
            int one = listOne==null? 0:listOne.val;
            int two = listTwo==null? 0:listTwo.val;
            int total = one + two + next;
            if (total > 9) {
                next = 1;
                total = total-10;
            }else{
                //没有进位将next归0
                next=0;
            }
            listOne.val = total;
            //如果存在进位，并且ListOne的next为null，则分配空间用来存储下一位的结果
            //先移位，用于判断是否进行下一位计算
            listTwo = listTwo==null? null:listTwo.next;
            if ((listOne.next == null&&next!=0)||(listOne.next==null&&listTwo!=null)) {
                listOne.next = new ListNode(0);
            }
            //移位
            listOne = listOne==null? null:listOne.next;
        }
        return res;
    }

## 无重复字符的最长子串
题目描述如下：

	//给定一个字符串，请你找出其中不含有重复字符的 最长子串 的长度。
	//
	// 示例 1:
	//
	// 输入: "abcabcbb"
	//输出: 3
	//解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
	//
	//
	// 示例 2:
	//
	// 输入: "bbbbb"
	//输出: 1
	//解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
	//
	//
	// 示例 3:
	//
	// 输入: "pwwkew"
	//输出: 3
	//解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
	//     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
	//
	// Related Topics 哈希表 双指针 字符串 Sliding Window
	// 👍 4310 👎 0

### 解法1
暴力法，通过遍历每个子串，得到最长的子串。代码如下：

    //暴力法：最后一个测试用例没有通过，最后一个测试用例耗时14秒
    public int lengthOfLongestSubstring0(String s) {
        if(null==s||s.equals("")){
            return 0;
        }
        //准备遍历字符串，提取没有重复字母的子串的长度，只保留最长的子串的长度
        int max=0;
        char[] source=s.toCharArray();
        int first,last;
        for (int j = 0; j < source.length; j++) {
            //双指针，指向子串的首末
            first=j;
            last=first;
            //hash表，值为1表示出现过此字母，值为0表示没有出现
            HashMap<Character, Integer> chars = new HashMap<>(source.length);

            for (int i = j; i < source.length; i++) {
                //如果当前字母没有出现过，则将last指针往后移动一位,并且hash表记录
                if(chars.get(source[i])==null){
                    chars.put(source[i],1);
                    last++;
                }else{
                    //如果出现过，则代表此时需要提取子串
                    int length = new String(source, first, last - first).length();
                    if(length>max){
                        max=length;
                    }
                    //重新设定first和last
                    first=last;
                    //因为for循环会i++，所以为了下一轮的开始，i需要--
                    i--;
                    //清空hash表
                    chars=new HashMap<>(source.length);
                }
            }
            //如果last到了字符串的最后都没有哈希命中，则代表从first到last也是一个没有重复的子串
            if(last==source.length){
                int length = new String(source, first, last - first).length();
                if(length>max){
                    max=length;
                }
            }
        }
        return max;
    }

### 解法2
使用Sliding Window（滑动窗口）得到无重复的最长子串的长度。代码如下：

    //滑动窗口，最后一个测试用例耗时：1毫秒
    public int lengthOfLongestSubstring(String s) {
        if (s.length()==0) return 0;
        //hash表，用于存放已存在最新字符的索引
        HashMap<Character, Integer> map = new HashMap<>();
        int max = 0;
        int left = 0;
        for(int i = 0; i < s.length(); i ++){
            if(map.containsKey(s.charAt(i))){
                //有可能重复的字符在窗口的左边，所以需要判断是否是窗口左边的字符
                //如果重复的字符是在窗口左边（不在窗口内），就不需要移动窗口的left
                left=Math.max(left,map.get(s.charAt(i))+1);
            }
            map.put(s.charAt(i),i);
            max = Math.max(max,i-left+1);
        }
        return max;
    }

## 寻找两个正序数组的中位数
题目表述如下：

	//给定两个大小为 m 和 n 的正序（从小到大）数组 nums1 和 nums2。
	//
	// 请你找出这两个正序数组的中位数，并且要求算法的时间复杂度为 O(log(m + n))。
	//
	// 你可以假设 nums1 和 nums2 不会同时为空。
	//
	//
	//
	// 示例 1:
	//
	// nums1 = [1, 3]
	//nums2 = [2]
	//
	//则中位数是 2.0
	//
	//
	// 示例 2:
	//
	// nums1 = [1, 2]
	//nums2 = [3, 4]
	//
	//则中位数是 (2 + 3)/2 = 2.5
	//
	// Related Topics 数组 二分查找 分治算法
	// 👍 3177 👎 0

### 解法1
脑袋简单的我又想到了暴力法=-=''''，思路很简单，就是将两个数组归并，排序，然后直接得到中位数。代码如下（使用Java提供的工具类归并排序）：

    //暴力法：先归并数组，然后求中位数
    public double findMedianSortedArrays1(int[] nums1, int[] nums2) {
        //准备合并两个数组
        int[] res = new int[nums1.length + nums2.length];
        System.arraycopy(nums1,0,res,0,nums1.length);
        System.arraycopy(nums2,0,res,nums1.length,nums2.length);
        //将数组排序
        Arrays.sort(res);
        //找出中位数
        boolean isDouble=res.length%2==0;
        int mid=res.length/2;
        return isDouble? (res[mid]+res[mid-1])/2.0:res[mid];
    }

### 解法2
改良版暴力法~~就是把上面解法1的归并排序用自己的算法实现，代码如下：

    //暴力法：先归并数组，然后求中位数（自己实现归并排序）
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {

        int i=0,j=0,count=0;
        int m=nums1.length;
        int n=nums2.length;

        //准备合并两个数组
        int[] res = new int[m+n];

        while(count!=(m+n)){
            //如果第一个数组已经遍历完啦,则直接将第二个数组剩余的添加到res后面（前提是原本两个数组就是有序的）
            if(i==m){
                while(j!=n){
                    res[count++]=nums2[j++];
                }
                break;
            }
            //如果第二个数组遍历完啦
            if(j==n){
                while(i!=m){
                    res[count++]=nums1[i++];
                }
                break;
            }
            int val1 = nums1[i];
            int val2 = nums2[j];
            res[count++]=val1<=val2? nums1[i++]:nums2[j++];
        }

        //找出中位数
        boolean isDouble=res.length%2==0;
        int mid=res.length/2;
        return isDouble? (res[mid]+res[mid-1])/2.0:res[mid];
    }