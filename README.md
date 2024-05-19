# Omochi

Omochi is a CLI tool to support Ruby on Rails development with RSpec. It detects methods uncovered by tests and prints the test code generated by LLM.

There are two advantages of using Omochi. We can make sure every method is covered by tests with GitHub Actions support. We can also generate spec files for uncovered methods so that we can reduce time to write them manually.

## Installation

```
$ gem specific_install -l https://github.com/mikik0/omochi.git
```

## Usage

```
$ omochi --help
Commands:
  omochi help [COMMAND]     # Describe available commands or one specific command
  omochi verify local_path  # verify spec created for all of new methods and functions
```

## Commands
### verify

When you execute Omochi locally, you can confirm that you have spec coverage for uncommitted diff.

```
# Local
$ omochi verify local_path
```

You can skip check with ignore comment.

```ruby
#omochi:ignore:
```

### --create option

You can generate spec files with LLM by giving `-c` or `--create` option. It outputs to STDOUT.

```
# Generate spec files
$ omochi verify local_path -c
# Or
$ omochi verify local_path --create
```

### --github option

With `-h` option you can check uncovered methods against given Pull Request with GitHub Actions.
It uses `gh pr diff` command internally.

```
$ omochi verify local_path -h
# Or
$ omochi verify local_path --github
```

## Samples

```
$ omochi verify -c
"Verify File List: [\"lib/omochi/cli.rb\", \"lib/omochi/util.rb\"]"
"specファイルあり"
===================================================================
verifyのテストを以下に表示します。
require 'rspec'

describe 'exit_on_failure?' do
  it 'returns true' do
    expect(exit_on_failure?).to eq(true)
  end
end
======= RESULT: lib/omochi/cli.rb =======
- exit_on_failure?
"specファイルなし"
======= RESULT: lib/omochi/util.rb =======
- local_diff_path
- github_diff_path
- remote_diff_path
- get_ast
- dfs
- find_spec_file
- get_pure_function_name
- dfs_describe
- print_result
- get_ignore_methods
- create_spec_by_bedrock
===================================================================
lib/omochi/util.rbのテストを以下に表示します。
require "spec_helper"

describe "local_diff_path" do
  it "returns array of diff paths from git" do
    allow(Open3).to receive(:capture3).with("git diff --name-only", any_args).and_return(["path1", "path2"], "", double(success?: true))
    expect(local_diff_path).to eq(["path1", "path2"])
  end

  it "returns empty array if git command fails" do
    allow(Open3).to receive(:capture3).with("git diff --name-only", any_args).and_return("", "error", double(success?: false))
    expect(local_diff_path).to eq([])
  end
end

describe "github_diff_path" do
  it "returns array of diff paths from gh" do
    allow(Open3).to receive(:capture3).with("gh pr diff --name-only", any_args).and_return(["path1", "path2"], "", double(success?: true))
    expect(github_diff_path).to eq(["path1", "path2"])
  end

  it "returns empty array if gh command fails" do
    allow(Open3).to receive(:capture3).with("gh pr diff --name-only", any_args).and_return("", "error", double(success?: false))
    expect(github_diff_path).to eq([])
  end
end

describe "remote_diff_path" do
  it "returns array of diff paths from remote" do
    allow(Open3).to receive(:capture3).with(/git diff --name-only .*${{ github\.sha }}/, any_args).and_return(["path1", "path2"], "", double(success?: true))
    expect(remote_diff_path).to eq(["path1", "path2"])
  end

  it "returns empty array if git command fails" do
    allow(Open3).to receive(:capture3).with(/git diff --name-only .*${{ github\.sha }}/, any_args).and_return("", "error", double(success?: false))
    expect(remote_diff_path).to eq([])
  end
end

describe "get_ast" do
  it "returns AST for given file" do
    allow(File).to receive(:read).with("file.rb").and_return("code")
    allow(Parser::CurrentRuby).to receive(:parse_with_comments).with("code").and_return(["ast"], ["comments"])
    expect(get_ast("file.rb")).to eq([{ast: "ast", filename: "file.rb"}])
  end
end

describe "dfs" do
  let(:node) { double(:node, type: :def, children: [double(:child, children: ["name"])]) }
  let(:result) { {} }

  it "traverses node and captures def names" do
    dfs(node, "file.rb", result)
    expect(result).to eq({"name" => "def name\nend"})
  end
end

describe "find_spec_file" do
  before do
    allow(File).to receive(:exist?).with("spec/app/file_spec.rb").and_return(true)
  end

  it "returns spec file path if exists" do
    expect(find_spec_file("app/file.rb")).to eq("spec/app/file_spec.rb")
  end

  it "returns nil if spec file does not exist" do
    allow(File).to receive(:exist?).with("spec/app/file_spec.rb").and_return(false)
    expect(find_spec_file("app/file.rb")).to be_nil
  end
end

# similarly test other functions
```

## Contributing

Bug reports and Pull Requests are accepted on GitHub (https://github.com/mikik0/omochi).
This project should be a safe place for collaboration.

### Design

1. `git diff` を取得する。--github オプションでは、`gh pr diff` を取得する
2. 差分のあったファイルの中から `.rb` だけを全て取得する
3. `.rb` ファイルを parser gem を用いて、抽象構文木(AST) にパースする
4. 取得したASTに対して、深さ優先探索(DFS)を用いて、全てのメソッドを取得する
5. 取得した全てのメソッドに対応するSpecがあるかどうかを確認するため、対応するSpecファイルを取得する
6. 取得したSpecファイルでも同様に、parser gem を用いて、抽象構文木(AST) にパースする
7. 取得したSpecファイルのASTにおいても同様に、深さ優先探索(DFS)を用いる。　describe に対応するメソッドが記述されているかを確認する。ここでは、 describe に、テストしたいメソッドのメソッド名が入っていることを前提としている。
8. Specが実装されていないメソッドが存在すれば出力し、異常終了する。Specが実装されていないメソッドが存在しなければ正常終了する。
9. --create オプションでは、Specが実装されていないメソッドに対するSpecの雛形を生成する。この際には、Amazon Bedrockを用いる。

### Development

#### Requirement

- ruby3.x
- AWS Credentials (for `--create`)

```
git clone https://github.com/mikik0/omochi.git
bundle install
bundle exec bin/omochi verify
```
