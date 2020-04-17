# Proposal of Proc#using



株式会社ネットワーク応用通信研究所

前田　修吾

## 背景

Refinementsはもともと特定のブロックに対して、メソッド側がRefinementsを有効にする機能を持っていたが、諸般の事情により機能が削除された

## ブロックレベルでRefinementsを有効にしたい理由

* 有効範囲を狭くすることで、より大胆な拡張が可能になる
* 主な用途はDSL

## 例1: RSpec

```ruby
it "should generate a 1 10% of the time (plus/minus 2%)" do
  result.occurences_of(1).should be_greather_than_or_equal_to(980)
  result.occurences_of(1).should be_less_than_or_equal_to(1020)
end
```

## 例2: ActiveRecord

```ruby
User.where { :name == 'matz' }
User.where { :age >= 18 }
```

## 先行事例

* Feature #12086: using: option for instance_eval etc.
    * https://bugs.ruby-lang.org/issues/12086

## using: option for instance_eval etc.

```ruby
module FixnumDivExt
  refine Fixnum do
    def /(other)
      quo(other)
    end
  end
end

p 1 / 2 #=> 0
instance_eval(using: FixnumDivExt) do
  p 1 / 2 #=> (1/2)
end
p 1 / 2 #=> 0
```

## なぜinstance_eval等の拡張として提案したか

* DSLの実装に便利だから

## Feature #12086の問題点

* スレッド安全性
* instance_execなどのサポートがない
* Refinementsの暗黙的有効化

## スレッド安全性

* 同じブロックが異なるRefinementsが有効な状態で実行される
* メソッドキャッシュの実装が困難

## instance_execなどのサポートがない

* instance_execは任意の引数をブロックパラメータに受け渡す
* 引数へのオプション追加が困難

## Refinementsの暗黙的有効化

* ユーザがコードを理解しづらくなるのではないかという懸念

## 新しい提案

* Feature #16461: Proc#using
    * https://bugs.ruby-lang.org/issues/16461

## Proc#using

```ruby
module IntegerDivExt
  refine Integer do
    def /(other)
      quo(other)
    end
  end
end

def instance_eval_with_integer_div_ext(obj, &block)
  block.using(IntegerDivExt) # blockが表すブロックでIntegerDivExtをusing
  obj.instance_eval(&block)
end

# Proc#usingを適用するブロックを書く場所で必要
using Proc::Refinements

p 1 / 2 #=> 0
instance_eval_with_integer_div_ext(1) do
  p self / 2 #=> (1/2)
end
p 1 / 2 #=> 0
```

## スレッド安全性

* Procオブジェクト単位ではなくブロック単位でRefinementsを有効化

## instance_execのサポート

* Proc#usingを呼んだ後でinstance_execに渡せば良い

## 明示的有効化

* Proc::Refinementsという特殊なモジュールをusingすることが必要
    * JRubyでの実装を簡単にするためでもある
* `using Proc::Refinements` をファイルの先頭などに書いておく
* 実際に有効になるRefinementsは、ブロック毎に何がProc#usingかされるかで変わる

## Proof of Concept実装

* For CRuby: https://github.com/shugo/ruby/pull/2
* For JRuby: https://github.com/shugo/jruby/pull/1

## 残課題

* ブロックが一度でも実行されたら新しいモジュールをProc#usingで追加できないようにしたい
* CRubyで違うクラスに対して同一ブロックでclass_evalした時にバグがあるのを修正したい
* MVM/Ractorでの挙動がどうあるべきか

## まとめ

* DSLを書きやすくするために先行事例の課題を解決するProc#usingを提案した
