
首先我们要腾出数组里第一个元素的位置，把所有的元素向右移动一位。我们 可以循环数组中的元素，从最后一位+1(长度)开始，将其对应的前一个元素的值赋给它，依次 处理，最后把我们想要的值赋给第一个位置(-1)上。