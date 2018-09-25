perf record -e cpu-clock --call-graph dwarf -p pid -o perf.data.hotkey.noptr sleep 30
perf diff perf.data.base perf.data.hotkey -d redis-server
perf report --call-graph  -G -i input

perf report:
* Self: 是函数自身的耗时
* Children: 本身的耗时加上所有子函数的耗时(子函数的耗时应该是包含子函数调用的函数的耗时)，可以认为是该函数的总耗时
