## 1. 快速排序

```go
func swap(arr []int, i int, j int) {
	tmp := arr[i]
	arr[i] = arr[j]
	arr[j] = tmp
}

func moveToMid(arr []int, start int, end int) int {
	i, j := start, end-1
	for i < j {
		for i < j && arr[i] <= arr[j] {
			j--
		}
		swap(arr, i, j)
		for i < j && arr[i] <= arr[j] {
			i++
		}
		swap(arr, i, j)
	}
	return i
}

func helper(arr []int, start int, end int) {
	if start >= end {
		return
	}
	if start+1 == end {
		return
	}
	mid := moveToMid(arr, start, end)
	helper(arr, start, mid)
	helper(arr, mid+1, end)
}

func quickSort(arr []int) {
	helper(arr, 0, len(arr))
}
```

循环不变量法快速排序：

```go
func quickSort(nums []int) {
	var helper func(nums []int, left int, right int)
	helper = func(nums []int, left, right int) {
		if left == right || left+1 == right {
			return
		}
		// x = nums[left]
		// allin [left, p1) < x
		// allin [p1, i) = x
		// allin [p2, right) > x
		x := nums[left]
		p1, i, p2 := left, left, right
		for i < p2 {
			if nums[i] < x {
				swap(nums, i, p1)
				p1++
				i++
			} else if nums[i] == x {
				i++
			} else {
				// nums[i] > x
				p2--
				swap(nums, i, p2)
			}
		}
		helper(nums, left, p1)
		helper(nums, p2, right)
	}
	helper(nums, 0, len(nums))
}
```

## 2. 归并排序

```go
func mergeOrderedList(arr []int, start int, mid int, end int) {
	tmpArr := make([]int, mid-start)
	copy(tmpArr, arr[start:mid])
	i := 0
	j := mid
	o := start
	for i < len(tmpArr) && j < end {
		if tmpArr[i] <= arr[j] {
			arr[o] = tmpArr[i]
			o++
			i++
		} else {
			arr[o] = arr[j]
			o++
			j++
		}
	}
	for i < len(tmpArr) {
		arr[o] = tmpArr[i]
		o++
		i++
	}
	for j < end {
		arr[o] = arr[j]
		o++
		j++
	}
}

func helper(arr []int, start int, end int) {
	if start >= end {
		return
	}
	if start+1 == end {
		return
	}
	mid := (start + end) / 2
	helper(arr, start, mid)
	helper(arr, mid, end)
	mergeOrderedList(arr, start, mid, end)
}

func mergeSort(arr []int) {
	helper(arr, 0, len(arr))
}
```

