<!--VITE PLUS START-->

# Using Vite+, the Unified Toolchain for the Web

This project is using Vite+, a unified toolchain built on top of Vite, Rolldown, Vitest, tsdown, Oxlint, Oxfmt, and Vite Task. Vite+ wraps runtime management, package management, and frontend tooling in a single global CLI called `vp`. Vite+ is distinct from Vite, and it invokes Vite through `vp dev` and `vp build`. Run `vp help` to print a list of commands and `vp <command> --help` for information about a specific command.

Docs are local at `node_modules/vite-plus/docs` or online at https://viteplus.dev/guide/.

## Review Checklist

- [ ] Run `vp install` after pulling remote changes and before getting started.
- [ ] Run `vp check` and `vp test` to format, lint, type check and test changes.
- [ ] Check if there are `vite.config.ts` tasks or `package.json` scripts necessary for validation, run via `vp run <script>`.
- [ ] If setup, runtime, or package-manager behavior looks wrong, run `vp env doctor` and include its output when asking for help.

<!--VITE PLUS END-->

## Books Reference

- `books/project-management-as-dandori`
  - 参照元 (Cosense): [Post.ダンドリとしてのプロジェクトマネジメント《Project_Management_as_Dandori》](https://scrapbox.io/la-ekstera-cerbo/Post.%E3%83%80%E3%83%B3%E3%83%89%E3%83%AA%E3%81%A8%E3%81%97%E3%81%A6%E3%81%AE%E3%83%97%E3%83%AD%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%83%9E%E3%83%8D%E3%82%B8%E3%83%A1%E3%83%B3%E3%83%88%E3%80%8AProject_Management_as_Dandori%E3%80%8B)
  - 参照方法: curl 等の直接アクセスではなく、`cosense` コマンド（例: `cosense browsePage <url>`）を使用すること。

