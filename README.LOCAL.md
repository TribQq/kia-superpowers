Это моя личная версия: локальная ветка `kia-superpowers`,
личный repo `git@github.com:TribQq/kia-superpowers.git`,
оригинал `git@github.com:obra/superpowers.git`.

Remotes:
- `upstream` - оригинальный репозиторий, оттуда только fetch/rebase.
- `origin` - мой личный GitHub, туда пушится рабочая ветка.

Обновить ветку из оригинала:

```bash
git fetch upstream
git switch kia-superpowers
git rebase upstream/main
```

Запушить ветку в мой репозиторий:

```bash
git push origin kia-superpowers
```

Если создал другой личный репозиторий на GitHub, заменить адрес `origin`:

```bash
git remote set-url origin git@github.com:YOUR_USER/YOUR_NEW_REPO.git
git remote -v
```

Потом запушить текущую ветку в новый личный репозиторий:

```bash
git push origin kia-superpowers
```

Если после rebase Git откажется пушить из-за переписанной истории:

```bash
git push origin kia-superpowers --force-with-lease
```

Не использовать `-u` (`--set-upstream`) для `kia-superpowers`: pull должен
оставаться явным через `upstream/main`.
