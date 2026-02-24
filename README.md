# ante

## 1. 作業を始める前（毎回）

まず main を最新にします。

```bash
git checkout main
git pull origin main
```

---

## 2. 作業用ブランチを作る（作業ごと）

ブランチ名は `feature/内容` がおすすめです。

```bash
git checkout -b feature/xxxx
```

例：

```bash
git checkout -b feature/login
```

---

## 3. commit（こまめに）

```bash
git add .
git commit -m "何をしたかを短く書く"
```

例：

```bash
git add .
git commit -m "ログイン画面を追加"
```

---

## 4. push（GitHubに上げる）

```bash
git push origin feature/xxxx
```

例：

```bash
git push origin feature/login
```

---

## 5. Pull Request（PR）を作る（GitHub上）

GitHubでPRを作成します。

- base: main
- compare: feature/xxxx

レビューしてもらってOKなら **Merge** します。

---

## 6. PRがmainにマージされた後（毎回）

自分のPCの main も最新にします。

```bash
git checkout main
git pull origin main
```

---

## 今どのブランチにいるか確認

```bash
git branch
```

---

## ルール（重要）

- mainに直接pushしない
- 必ずブランチを作る
- PRを作ってからmainに入れる
- 作業前は main を pull して最新にする
