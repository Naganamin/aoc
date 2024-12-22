# Advent of Code 2024 - Day 15

```elixir
Mix.install([
  :kino_aoc
])
```

## Introduction

[2024 - Day 15](https://adventofcode.com/2024/day/15)

## Puzzle

<!-- livebook:{"attrs":"eyJhc3NpZ25fdG8iOiJwdXp6bGVfaW5wdXQiLCJkYXkiOiIxNSIsInNlc3Npb25fc2VjcmV0IjoiQU9DX1NFU1NJT04iLCJ5ZWFyIjoiMjAyNCJ9","chunks":null,"kind":"Elixir.KinoAOC.HelperCell","livebook_object":"smart_cell"} -->

```elixir
{:ok, puzzle_input} =
  KinoAOC.download_puzzle("2024", "15", System.fetch_env!("LB_AOC_SESSION"))
```

## Parser

### Code - Parser

```elixir
defmodule Parser do
  def parse(input) do
    input
    |> String.split("\n", trim: true)
    |> Enum.chunk_by(&String.contains?(&1, "<"))
    |> then(fn [map = [head | _], steps] ->
      max_x = String.length(head) - 1
      max_y = Enum.count(map) - 1

      {
        map
        |> Enum.with_index()
        |> Enum.flat_map(fn {line, y} ->
          line
          |> String.graphemes()
          |> Enum.with_index()
          |> Enum.filter(fn {char, _} -> char != "." end)
          |> Enum.map(fn {char, x} -> {{x, y}, char} end)
        end)
        |> Map.new(),
        steps
        |> Enum.flat_map(&String.graphemes/1)
        |> Enum.map(fn
          "<" -> {-1, 0}
          ">" -> {1, 0}
          "^" -> {0, -1}
          "v" -> {0, 1}
        end),
        max_x,
        max_y
      }
    end)
  end
end
```

### Tests - Parser

```elixir
ExUnit.start(autorun: false)

defmodule ParserTest do
  use ExUnit.Case, async: true
  import Parser

  @input """
  ####
  #.O#
  #@##
  ####

  ^v<>
  """
  @expected {%{
               {1, 2} => "@",
               {2, 1} => "O",
               {2, 2} => "#",
               {0, 2} => "#",
               {0, 3} => "#",
               {1, 0} => "#",
               {1, 3} => "#",
               {2, 0} => "#",
               {2, 3} => "#",
               {3, 0} => "#",
               {3, 1} => "#",
               {3, 2} => "#",
               {3, 3} => "#",
               {0, 0} => "#",
               {0, 1} => "#"
             }, [{0, -1}, {0, 1}, {-1, 0}, {1, 0}], 3, 3}

  test "parse test" do
    actual = parse(@input)
    assert actual == @expected
  end
end

ExUnit.run()
```

## Shared

```elixir
defmodule Shared do
  def print_map(map, max_x, max_y) do
    0..max_y
    |> Enum.each(fn y ->
      line =
        0..max_x
        |> Enum.map(fn x ->
          Map.get(map, {x, y}, ".")
        end)
        |> Enum.join()

      IO.inspect(line)
    end)
  end
end
```

<!-- livebook:{"branch_parent_index":2} -->

## Part One

### Code - Part 1

```elixir
defmodule PartOne do
  def solve(input) do
    IO.puts("--- Part One ---")
    IO.puts("Result: #{run(input)}")
  end

  def run(input) do
    {map, steps, max_x, max_y} = Parser.parse(input)

    {start, _} =
      map
      |> Enum.find(fn {_, char} -> char == "@" end)

    map
    |> move(max_x, max_y, start, steps)
    |> Enum.map(fn
      {{x, y}, "O"} -> x + 100 * y
      _ -> 0
    end)
    |> Enum.sum()
  end

  def move(map, _, _, _, []), do: map

  def move(map, max_x, max_y, curr = {x, y}, [{dx, dy} | steps]) do
    next = {x + dx, y + dy}

    case Map.get(map, next) do
      "#" ->
        move(map, max_x, max_y, curr, steps)

      nil ->
        map
        |> Map.delete(curr)
        |> Map.put(next, "@")
        |> move(max_x, max_y, next, steps)

      "O" ->
        {max, d, dir} = if dx == 0, do: {max_y, dy, :y}, else: {max_x, dx, :x}

        case to_push(map, max, next, d, dir) do
          [] ->
            move(map, max_x, max_y, curr, steps)

          pushes ->
            map
            |> Map.delete(curr)
            |> then(fn map ->
              pushes
              |> Enum.reduce(map, fn {pos, char}, map ->
                Map.put(map, pos, char)
              end)
            end)
            |> move(max_x, max_y, next, steps)
        end
    end
  end

  defp to_push(map, max_y, start = {x, start_y}, d, :y) do
    Stream.iterate(start_y + d, &(&1 + d))
    |> Stream.take_while(&(&1 >= 0 and &1 <= max_y))
    |> Enum.reduce_while([{start, "@"}], fn y, pushes ->
      curr = {x, y}

      case Map.get(map, curr) do
        "O" -> {:cont, [{curr, "O"} | pushes]}
        "#" -> {:halt, []}
        nil -> {:halt, [{curr, "O"} | pushes]}
      end
    end)
  end

  defp to_push(map, max_x, start = {start_x, y}, d, :x) do
    Stream.iterate(start_x + d, &(&1 + d))
    |> Stream.take_while(&(&1 >= 0 and &1 <= max_x))
    |> Enum.reduce_while([{start, "@"}], fn x, pushes ->
      curr = {x, y}

      case Map.get(map, curr) do
        "O" -> {:cont, [{curr, "O"} | pushes]}
        "#" -> {:halt, []}
        nil -> {:halt, [{curr, "O"} | pushes]}
      end
    end)
  end
end
```

### Tests - Part 1

```elixir
ExUnit.start(autorun: false)

defmodule PartOneTest do
  use ExUnit.Case, async: true
  import PartOne

  @input """
  ##########
  #..O..O.O#
  #......O.#
  #.OO..O.O#
  #..O@..O.#
  #O#..O...#
  #O..O..O.#
  #.OO.O.OO#
  #....O...#
  ##########

  <vv>^<v^>v>^vv^v>v<>v^v<v<^vv<<<^><<><>>v<vvv<>^v^>^<<<><<v<<<v^vv^v>^
  vvv<<^>^v^^><<>>><>^<<><^vv^^<>vvv<>><^^v>^>vv<>v<<<<v<^v>^<^^>>>^<v<v
  ><>vv>v^v^<>><>>>><^^>vv>v<^^^>>v^v^<^^>v^^>v^<^v>v<>>v^v^<v>v^^<^^vv<
  <<v<^>>^^^^>>>v^<>vvv^><v<<<>^^^vv^<vvv>^>v<^^^^v<>^>vvvv><>>v^<<^^^^^
  ^><^><>>><>^^<<^^v>>><^<v>^<vv>>v>>>^v><>^v><<<<v>>v<v<v>vvv>^<><<>^><
  ^>><>^v<><^vvv<^^<><v<<<<<><^v<<<><<<^^<v<^^^><^>>^<v^><<<^>>^v<v^v<v^
  >^>>^v>vv>^<<^v<>><<><<v<<v><>v<^vv<<<>^^v^>^^>>><<^v>>v^v><^^>>^<>vv^
  <><^^>^^^<><vvvvv^v<v<<>^v<v>v<<^><<><<><<<^^<<<^<<>><<><^^^>^^<>^>v<>
  ^^>vv<^v^v<vv>^<><v<^v>^^^>>>^^vvv^>vvv<>>>^<^>>>>>^<<^v>^vvv<>^<><<v>
  v^^>>><<^^<>>^v^<v^vv<>v^<<>^<^v^v><^<<<><<^<v><v<>vv>>v><v^<vv<>v^<<^
  """
  @expected 10092

  test "part one" do
    actual = run(@input)
    assert actual == @expected
  end
end

ExUnit.run()
```

### Solution - Part 1

```elixir
PartOne.solve(puzzle_input)
```

<!-- livebook:{"branch_parent_index":2} -->

## Part Two

### Code - Part 2

```elixir
defmodule PartTwo do
  def solve(input) do
    IO.puts("--- Part Two ---")
    IO.puts("Result: #{run(input)}")
  end

  def run(input) do
    {map, steps, max_x, max_y} = Parser.parse(input)

    map = x2(map)
    max_x = max_x * 2 + 1

    {start, _} =
      map
      |> Enum.find(fn {_, char} -> char == "@" end)

    map
    |> move(max_x, max_y, start, steps)
    |> Enum.map(fn
      {{x, y}, "["} -> x + 100 * y
      _ -> 0
    end)
    |> Enum.sum()
  end

  def move(map, _, _, _, []), do: map

  def move(map, max_x, max_y, curr = {x, y}, [{dx, 0} | steps]) do
    next = {x + dx, y}

    case Map.get(map, next) do
      "#" ->
        move(map, max_x, max_y, curr, steps)

      nil ->
        map
        |> Map.delete(curr)
        |> Map.put(next, "@")
        |> move(max_x, max_y, next, steps)

      char when char in ["[", "]"] ->
        case push_left_right(map, max_x, next, dx) do
          [] ->
            move(map, max_x, max_y, curr, steps)

          pushes ->
            map
            |> Map.delete(curr)
            |> then(fn map ->
              pushes
              |> Enum.reduce(map, fn {pos, char}, map ->
                Map.put(map, pos, char)
              end)
            end)
            |> move(max_x, max_y, next, steps)
        end
    end
  end

  def move(map, max_x, max_y, curr = {x, y}, [{0, dy} | steps]) do
    next = {x, y + dy}

    case Map.get(map, next) do
      "#" ->
        move(map, max_x, max_y, curr, steps)

      nil ->
        map
        |> Map.delete(curr)
        |> Map.put(next, "@")
        |> move(max_x, max_y, next, steps)

      char when char in ["[", "]"] ->
        case push_up_down(map, next, dy, MapSet.new(), []) do
          {[], _} ->
            move(map, max_x, max_y, curr, steps)

          {pushes, _} ->
            {back, sort_modifier} = if dy == 1, do: {-1, -1}, else: {1, 1}

            pushes
            |> List.flatten()
            |> Enum.sort_by(fn {_, y} -> y * sort_modifier end)
            |> Enum.reduce(map, fn curr = {x, y}, map ->
              prev = {x, y + back}
              prev_char = Map.get(map, prev)

              map
              |> Map.put(curr, prev_char)
              |> Map.delete(prev)
            end)
            |> Map.delete(curr)
            |> Map.put(next, "@")
            |> move(max_x, max_y, next, steps)
        end
    end
  end

  defp push_left_right(map, max_x, start = {start_x, y}, dx) do
    Stream.iterate(start_x + dx, &(&1 + dx))
    |> Stream.take_while(&(&1 >= 1 and &1 <= max_x))
    |> Enum.reduce_while([{start, "@"}], fn x, pushes ->
      curr = {x, y}

      case Map.get(map, curr) do
        "[" ->
          {:cont, [{curr, "]"} | pushes]}

        "]" ->
          {:cont, [{curr, "["} | pushes]}

        "#" ->
          {:halt, []}

        nil ->
          case dx do
            1 -> {:halt, [{curr, "]"} | pushes]}
            -1 -> {:halt, [{curr, "["} | pushes]}
          end
      end
    end)
  end

  defp push_up_down(map, start = {x, y}, dy, visited, pushes) do
    box =
      [left, right] =
      case Map.get(map, start) do
        "[" -> [start, {x + 1, y}]
        "]" -> [{x - 1, y}, start]
      end

    if MapSet.member?(visited, left) or MapSet.member?(visited, right) do
      {pushes, visited}
    else
      visited = visited |> MapSet.put(left) |> MapSet.put(right)

      case box
           |> Enum.map(fn {x, y} ->
             next = {x, y + dy}
             {next, Map.get(map, next)}
           end) do
        [{_, "#"}, _] ->
          {[], visited}

        [_, {_, "#"}] ->
          {[], visited}

        [{left, nil}, {right, nil}] ->
          {[left, right | pushes], visited}

        [{left, nil}, {right, _}] ->
          push_up_down(map, right, dy, visited, [left, right | pushes])

        [{left, _}, {right, nil}] ->
          push_up_down(map, left, dy, visited, [left, right | pushes])

        [{left, _}, {right, _}] ->
          case push_up_down(map, left, dy, visited, [left, right | pushes]) do
            {[], visited} -> {[], visited}
            {pushes, visited} -> push_up_down(map, right, dy, visited, pushes)
          end
      end
    end
  end

  defp x2(map) do
    map
    |> Enum.flat_map(fn
      {{x, y}, "@"} -> [{{2 * x, y}, "@"}]
      {{x, y}, "O"} -> [{{2 * x, y}, "["}, {{2 * x + 1, y}, "]"}]
      {{x, y}, "#"} -> [{{2 * x, y}, "#"}, {{2 * x + 1, y}, "#"}]
    end)
    |> Map.new()
  end
end
```

### Tests - Part 2

```elixir
ExUnit.start(autorun: false)

defmodule PartTwoTest do
  use ExUnit.Case, async: true
  import PartTwo

  @input """
  ##########
  #..O..O.O#
  #......O.#
  #.OO..O.O#
  #..O@..O.#
  #O#..O...#
  #O..O..O.#
  #.OO.O.OO#
  #....O...#
  ##########

  <vv>^<v^>v>^vv^v>v<>v^v<v<^vv<<<^><<><>>v<vvv<>^v^>^<<<><<v<<<v^vv^v>^
  vvv<<^>^v^^><<>>><>^<<><^vv^^<>vvv<>><^^v>^>vv<>v<<<<v<^v>^<^^>>>^<v<v
  ><>vv>v^v^<>><>>>><^^>vv>v<^^^>>v^v^<^^>v^^>v^<^v>v<>>v^v^<v>v^^<^^vv<
  <<v<^>>^^^^>>>v^<>vvv^><v<<<>^^^vv^<vvv>^>v<^^^^v<>^>vvvv><>>v^<<^^^^^
  ^><^><>>><>^^<<^^v>>><^<v>^<vv>>v>>>^v><>^v><<<<v>>v<v<v>vvv>^<><<>^><
  ^>><>^v<><^vvv<^^<><v<<<<<><^v<<<><<<^^<v<^^^><^>>^<v^><<<^>>^v<v^v<v^
  >^>>^v>vv>^<<^v<>><<><<v<<v><>v<^vv<<<>^^v^>^^>>><<^v>>v^v><^^>>^<>vv^
  <><^^>^^^<><vvvvv^v<v<<>^v<v>v<<^><<><<><<<^^<<<^<<>><<><^^^>^^<>^>v<>
  ^^>vv<^v^v<vv>^<><v<^v>^^^>>>^^vvv^>vvv<>>>^<^>>>>>^<<^v>^vvv<>^<><<v>
  v^^>>><<^^<>>^v^<v^vv<>v^<<>^<^v^v><^<<<><<^<v><v<>vv>>v><v^<vv<>v^<<^
  """
  @expected 9021

  test "part two" do
    actual = run(@input)
    assert actual == @expected
  end
end

ExUnit.run()
```

### Solution - Part 2

```elixir
PartTwo.solve(puzzle_input)
```

<!-- livebook:{"offset":11661,"stamp":{"token":"XCP.pdxQxyVNLoXEhcnReo3BYSF0AyW-39Lr6pBCB-n6nGQXKWXxloJLavC2r1hL0VbTzalJL3zgYgHXBQpEMm9WBpekwoN0UEUB5R1zQ4A7aOcAImB5Ieg","version":2}} -->
