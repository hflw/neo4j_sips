Neo4j.Sips
==========

A simple Elixir wrapper around the Neo4j graph database REST API. It is aiming to help the Elixir developers explore [Neo4j](http://neo4j.com/developer/get-started/), and to eventually become the core of a future [Ecto](https://github.com/elixir-lang/ecto) adapter.

### Install

Edit mix.ex and add the `neo4j_sips` dependency to the `deps/1 `function:

    defp deps do
      [{:neo4j_sips, "~> 0.1"}]
    end

or Github:

    defp deps do
      [{:neo4j_sips, github: "florinpatrascu/neo4j_sips"}]
    end

Or, if you're using a local development copy:

    defp deps do
      [{:neo4j_sips, path: "../neo4j_sips"}]
    end

Then add the `neo4j_sips` dependency the applications list:

    def application do
      [applications: [:logger, :neo4j_sips]]
    end


Edit the `config/config.exs` and describe a Neo4j server endpoint, example:

    config :neo4j_sips, Neo4j,
      url: "http://localhost:7474",
      pool_size: 5,
      max_overflow: 2,
      timeout: 30

Run `mix do deps.get, deps.compile`

If your server requires basic authentication, add this to your config file:
      
      basic_auth: [username: "foo", password: "bar"]
      
Or:
      
      token_auth: "bmVvNGo6dGVzdA==" # if using an authentication token?!
  
### Example

With a minimalist setup configured as above, and a Neo4j server running, we can connect to the server and run some queries using Elixir’s interactive shell ([IEx](http://elixir-lang.org/docs/stable/iex/IEx.html)):

    $ cd <my_mix_project>
    $ iex -S mix
    Erlang/OTP 18 [erts-7.1] [source] [64-bit] ....

    Interactive Elixir (1.1.1) - press Ctrl+C to exit ....
    iex(1)> alias Neo4j.Sips, as: Neo4j

    iex(2)> cypher = """
      CREATE (n:Neo4jSips {title:'Elixir sipping from Neo4j', released:2015, 
        license:'MIT', neo4j_sips_test: true})
    """

    iex(3)> Neo4j.query(Neo4j.conn, cypher)
    {:ok, []}

    iex(4)> n = Neo4j.query!(Neo4j.conn, "match (n:Neo4jSips {title:'Elixir sipping from Neo4j'}) where n.neo4j_sips_test return n")
    [%{"n" => %{"license" => "MIT", "neo4j_sips_test" => true, "released" => 2015,
         "title" => "Elixir sipping from Neo4j"}}]
    iex(5)> 
    
## Model support

Using the Model, you can easily define your own Elixir modules like this:

```elixir
defmodule Person do
  use Neo4j.Sips.Model

  field :name, required: true
  field :email, required: true, unique: true, format: ~r/\b[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,4}\b/
  field :age, type: :integer
  field :doe_family, type: :boolean, default: false # used for testing
  field :neo4j_sips, type: :boolean, default: true

  validate_with :check_age

  relationship :FRIEND_OF, Person
  relationship :MARRIED_TO, Person

  def check_age(model) do
    if model.age == nil || model.age <= 0 do
      {:age, "model.validation.invalid_age"}
    end
  end
end

```

and use in various scenarios. Example from various tests file:

```elixir
assert {:ok, john} = Person.create(name: "John DOE", email: "john.doe@example.com",
                                   age: 30, doe_family: true,
                                   enable_validations: true)
assert john != nil
assert {:ok, jane} = Person.create(name: "Jane DOE", email: "jane.doe@example.com",
                                   age: 25, enable_validations: true, doe_family: true,
                                   married_to: john)
on_exit({john, jane}, fn ->
    assert :ok = Person.delete(john)
    assert :ok = Person.delete(jane)
  end)

...

# model find
test "find Jane DOE" do
  persons = Person.find!(name: "Jane DOE")
  assert length(persons) == 1

  person = List.first(persons)
  assert person.name == "Jane DOE"
  assert person.email == "jane.doe@example.com"
  assert person.age == 25
end

test "find John DOE" do
  persons = Person.find!(name: "John DOE")
  assert length(persons) == 1

  person = List.first(persons)
  assert person.name == "John DOE"
  assert person.email == "john.doe@example.com"
  assert person.age == 30
end

...

# serialization
Person.to_json(jane)  

# support for relationships
relationship_names = Person.metadata.relationships |> Enum.map(&(&1.name))
relationship_related_models = Person.metadata.relationships |> Enum.map(&(&1.related_model))
assert relationship_names == [:FRIEND_OF, :MARRIED_TO]
assert relationship_related_models == [Person, Person]

...

#support for validation
test "invalid mail format" do
  {:nok, nil, person} = Person.create(name: "John Doe", email: "johndoe.example.com", age: 30)
  assert Enum.find(person.errors[:email], &(&1 == "model.validation.invalid")) != nil
end

test "invalid age value" do
  {:nok, nil, person} = Person.create(name: "John Doe", email: "john.doe@example.com", age: -30)
  assert Enum.find(person.errors[:age], &(&1 == "model.validation.invalid_age")) != nil
end


## and more
```

For more examples, see the test suites.

## License
* Neo4jSips - MIT, check LICENSE file for more information.
* Neo4j - Dual free software/commercial license, see http://neo4j.org/
