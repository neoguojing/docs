# 技巧

## 根据目标改变变量名称
```
GO_GCFLAGS=""
build: $(TARGETS)

debug: GO_GCFLAGS="-l -N"
debug: $(TARGETS)

$(TARGETS): $(SRC)
	$(GO) build -gcflags $(GO_GCFLAGS) -ldflags '$(LDFLAGS)' -mod=vendor  $(TEST_FLAGS) $(project)/cmd/$@
```
