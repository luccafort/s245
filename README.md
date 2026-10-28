# s245

Ruby のおもしろいメソッド（`String#squeeze`、`String#succ`、`Integer#digits` など）を、[Spinel](https://github.com/matz/spinel)（Ruby → C）と [ccgo](https://pkg.go.dev/modernc.org/ccgo/v4)（C → Go）を連結して Go に移植するライブラリです。挙動は CRuby との差分テストで保証します。

設計の判断は [docs/adr](docs/adr) にまとめています。

- [ADR 0001: Spinel と ccgo を連結して Ruby のメソッドを Go に移植する](docs/adr/0001-spinel-c-go-pipeline.md)
