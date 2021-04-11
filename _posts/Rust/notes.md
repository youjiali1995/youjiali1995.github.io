## debug test
https://stackoverflow.com/questions/27269315/how-do-i-debug-a-failing-cargo-test-in-gdb

```
rust-gdb target/debug/deps/tikv-fbf40b8e31eaef97
break rust_panic
run --test test_extract_failure
```
