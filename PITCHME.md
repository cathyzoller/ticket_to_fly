---?image=assets/bg1.jpeg

---?image=assets/bg2.jpeg

Note: Over the past year and a half I've been working on a re-write of an application called Knitwiz - that allows users to design sweaters and auto generate the instructions using any pattern stitch plus their measurements.  Originally written in rails with a fair amount of jquery, the stack is now comprised of a phoenix api along with a client-side React app. It launched about a month ago, & is currently in beta - so if any of you would like to knit a sweater let me know after.  Right now though I'd like to talk a bit about three  "tools" (for lack of a better word) I found indispensible for this project, and my gradual  shift in thinking as I moved from Ruby to Elixir.  i.e. moving from an object-oriented approach to a functional one. (slide) How many of you are just beginning with Elixir/Phoenix?  Use as an api? Use React? (slide)

---?image=assets/bg3.jpeg

Note: The three tools are Images, Ecto.Multi and Recursion.  By the way, I'm using Phoenix in this project, but much of what I'll be saying is not specific to the phoenix platform. (slide)

---?image=assets/bg5.jpeg

Note: The primary hex package used for uploading images is Arc. As with the ruby gems Paperclip or Carrierwave, Arc works with imagemagick for thumbnail creation or other processing.  And that's about where the similarity ends.  Arc takes advantage of the Plug.Upload struct - more detail about that later. Once uploaded, the return is a standard tuple. (slide)

---?image=assets/bg6.jpeg

Note: In order to scope to a record, you'll also need Arc_Ecto, which provides a cast_attachment function

---

```elixir
def changeset(struct, params \\ %{}) do
  struct
  |> cast(params, [:name, :description, :user_id])
  |> cast_attachments(params, [:photo])
  |> validate_required([:name, :description, :user_id])
  |> unique_constraint(:name, message: "already taken")
end
```

@[1-3]
@[4]
@[1-7]

---

### Uploader

```elixir
# uploaders/photo
def acl(:original, _), do: :public_read

def storage_dir(_, {_, scope}) do
   "uploads/design/#{scope.id}"
end
```

@[4-6]
@[5]
@[2]

Note: couple of gotchas: (slide) First, scope.id not available on create. Workaround - create then update &/or use different changeset functions. (slide) Secondly, for public display of images hosted on AWS S3 add a public read acl.  Overall I thought it was much easier to push images up to s3 and recall them, than any process I've used before.  What's the process look like?

---

<img src="assets/fido-sweater.jpg" width="600px" />

Note: Working from frontend to backend, the process is as follows: the user wants to upload a design for a dog sweater.


---
### React Form Component

```javascript
handleImageChange(e) {
  e.preventDefault();

  const reader = new FileReader();
  const file = e.target.files[0];

  reader.onloadend = () => {
    this.setState({
      imagePreviewUrl: reader.result
    });
    this.setState({ design: { photo: file } });
  }

  reader.readAsDataURL(file)
}

handlePhotoUpdate() {
  this.props.setDesignField('photo', this.state.design.photo);
  this.props.submitPhotoUpdate(this.props.designId);
}
```

@[1-5]

@[7-12]

@[16-19]

Note: Javascript looks like this (slide) Using the FileReader construct makes the frontend image manipulation pretty straightforward.  (slide) Calling a function which calls the Phoenix endpoint

---

```javascript
export const updatePhoto = (id) => (
  (dispatch, getState) => {
    const { formData } = getState().designs;
    let form_data = new FormData();

    Object.keys(formData).forEach((key) => {
      if (formData[key] instanceof File) {
        form_data.append(`design[${key}]`, formData[key], formData[key].name);
      } else {
        form_data.append(`design[${key}]`, formData[key]);
      }
    });

    httpPostForm(`/api/v1/designs/${id}`, form_data)
    .then((resp) => {
```

@[4]

@[6-12]

@[14]


Note: We're using javascript's FormData, (slide) appending the data we want to send to the backend. (slide).  httpPostForm is just a utility function calls fetch with a POST method

---

### Endpoint Receives Image Data from a Client Application

---

### FormData objects must be sent in a POST request

#### update router.ex

* resources "/designs", DesignController, except: [:new, :edit]
* post "/designs/:id", DesignController, :update


Note: FormData objects require a POST request

---

### DesignController update action

---

```elixir
#[debug] Processing by Knitwiz.DesignController.update/2

parameters: %{
  "id" => "6",
  "design" => %{
    "description" => "doggie sweater",
    "name" => "Fido's Sweater",
    "photo" => %Plug.Upload{
      content_type: "image/png",
      filename: "fidos-sweater.png",
      path: "/var/folders/2s/fs..66/T//plug-1493/multipart-53892"
    },
    "supplies" => "yarn"
  }
}
```

@[8-12]

Note: controller has access to Plug.Upload where a Genserver process saves upload struct to a temp directory. After process dies the file moved to permanent home (either cdn or local store)

---

### SVG file manipulation

<img src="assets/small-dog-template.jpg" width="50%" height="50%" />


Note: original dog sweater template is copied inside s3 & pulled down to manipulate inside to further manipulate.(slide)

---

#### Elixir

```elixir
File.cp(source, destination, callback \\ fn _, _ -> true end)
# => {:ok} OR {:error, :reason}
```
#### OR Erlang

```elixir

:file.copy(source, destination, bytes_count \\ :infinity)
# => {:ok, :bytes_copied} OR {:error, :reason}
```
Note: To digress for a moment, you may also copy files using the File module


---

```xml
<svg width="640" height="480" xmlns="http://www.w3.org/2000/svg" xmlns:svg="http://www.w3.org/2000/svg">
 <g>
  <title>Layer 1</title>
  <path
    id="svg_6"
    stroke="#5656ff"
    d="m159.408943,208.251741c31.05263,-13 64.10526,-35
      87.15789,-60c62.94737,4.66667 128.89473,
      -9.66667 188.8421,14c59.94737,23.66667 19.62907,
      84 -49.55639,116l-96.74435,27c-90.45113,
      1.33333 -110.41354,-52.33333 -129.69925,-97z"
    stroke-linecap="null"
    stroke-linejoin="null"
    stroke-dasharray="null"
    stroke-width="5"
    fill="#ff56ff" />
 </g>
</svg>
```

@[7-11]

#### Dog Sweater Template SVG

Note: Here's the xml which represents the template.  Changing the path argument "d" will change the shape.

---

<img src="assets/big-dog-1.jpg" width="50%" height="50%" />

### Using d3 or similar library

Note: Once the template is overlayed onto a model, it can be manipulated with d3 or fabric.js (although fabric.js doesn't have react syntax baked in, it wasn't difficult to refactor)(slide)

---
<img src="assets/big-dog-2.jpg" width="50%" height="50%" />

### Save the Transformed File...

---

#### Using Nokogiri with Rails

```ruby
# AJAX POST -> updates svg path with Nokogiri

  def update_path
    if params[:id] && params[:svg_d_attr]
      @pattern = Pattern.find(params[:id].to_i)

      file_path = "#{Rails.root}/public/#{@pattern.image_url}"

      doc = Nokogiri::XML(File.read file_path)

      doc.css("path").first["d"] = params[:svg_d_attr]

      File.open(file_path,'w') {|f| doc.write_xml_to f}
    end
  end
```

@[7]

@[9-11]

@[13]

@[1-14]

Note:  get the file path (next), then set the css(next) then write to the file(next) - took a little over 2 seconds so make sure you have a loading spinner!

---

```elixir
  %HTTPoison.Response{body: body, status_code: status_code} = HTTPoison.get!(file_path)

  case status_code do
    200 ->
      tmp_path = "tmp/#{pattern_piece.id}/#{pattern_piece.pattern_name}.svg"
      File.mkdir("tmp/#{pattern_piece.id}")
      File.write!(tmp_path, body)

      case PatternPiece.transform_path(tmp_path, params["path"]) do
        {:ok, _} ->
          PatternImage.store({tmp_path, pattern_piece})

          %{status: 201, msg: "success"}
        {:error, _} ->
          %{status: 422, msg: "error in transform"}
      end
    _ ->
      %{status: 422, msg: "file not found"}
  end
```

@[1]
@[3-7]
@[9-11]

Note: s3 will return something whether or not the file exists(next). Switch on the status_code (next) & if a 200 response then write the file to a temporary directory, prior to transforming. (next).  If the transformation is successful, then replace the version on s3 with this one.  (slide)

---

## Floki

```elixir
def transform_path(svg_url, new_path) do
  File.open(svg_url, [:read, :write], fn(file) ->
    svg = IO.read(file, :all)
    |> Floki.attr("path", "d", fn _ ->  new_path end)
    |> prepend_xml
    |> Floki.raw_html

    File.write(svg_url, svg)
  end)
end

defp prepend_xml(list), do: [{:pi, "xml", [{"version", "1.0"}]} | list]
```

#### Floki.attr(tree, element, selector, mutation function)

Note: For the actual transformation we can use Floki.attr method. Floki works by creating a list of the xml or html elements. The attr function changes the attribute values of the elements matched by `selector` with the function `mutation` and returns the whole element tree.  Using elixir speeds the process up to 100 to 200 ms, and about 55 ms without latency. So even complex image manipulation is straightforward to implement and 10 times more performant.  Next up, ecto multi.

---

## Ecto.Multi transactions

#### transactions with pipes

---

### Ruby or Javascript

#### Conditionals

---

```Ruby
def complete_purchase(purchase, is_birthday, coupon)
  if !nil?(coupon)
    apply_discount(purchase, coupon)
  elsif is_birthday
    if purchase > 2000
      send_big_treat
    else
      send_little_treat
    end
  else
    say_thank_you
  end
end
```
@[1-3]

@[4-9]

@[2, 4, 10-12]

---

### Elixir

#### Guard clauses

---

```elixir
def complete_purchase(purchase, is_birthday, coupon) when is_nil(coupon) do
  case is_birthday do
    true -> send_treat(purchase)
    _ -> say_thank_you
  end
end
def complete_purchase(purchase, is_birthday, coupon) do
  apply_discount(purchase, coupon)
end

defp send_treat(purchase) when purchase > 20, do: send_big_treat
defp send_treat(purchase), do: send_little_treat

```
@[1, 6, 7-9]

@[1-6, 11-12]


Note: one way of writing with elixir

---

#### Refactor with Pipe operator

```elixir
def complete_purchase(purchase, is_birthday, coupon) do
  apply_coupon(coupon)
  |> check_birthday(is_birthday)
  |> send_treat(purchase)
end

defp apply_coupon(coupon) when is_nil(coupon), do: {:continue, "no coupon"}
defp apply_coupon(coupon) do
  calculate_discount
  # => {:stop, "discount applied"} OR {:stop, "error occurred"}
end

defp check_birthday({:stop, reason}, is_birthday), do: {:stop, reason}
defp check_birthday({:continue, reason}, is_birthday) when !is_birthday do
  say_thank_you
  {:stop, "no birthday"}
end
defp check_birthday({:continue, reason}, is_birthday) do
  {:continue, "has birthday"}
end

defp send_treat({:stop, reason}, purchase), do: {:stop, reason}
defp send_treat({:continue, reason}, purchase) when purchase > 2000, do: send_big_treat
defp send_treat({:continue, reason}, purchase), do: send_little_treat
defp send_treat(_, _), do: {:error, "Hmmmm....."}
```

@[1-5]

@[7-11]

@[13-20]

@[21-25]

Note: Refactor using pattern matching as well...Another use for pipes next

---


### Ecto Multi

```elixir
  def manage_stripe_charge(user, design_id, design_name, token, amount) do
    Multi.new
    |> Multi.run(:retrieve_customer, &retrieve_customer(&1, user.stripe_id, token))
    |> Multi.run(:update_user, &update_user(&1.retrieve_customer, user))
    |> Multi.run(:stripe_charge, &stripe_charge(&1.retrieve_customer, amount))
    |> Multi.run(:insert_order, &insert_order(&1.stripe_charge, user.id, design_id))
    |> Multi.run(:insert_charge, &insert_charge(&1.insert_order))
    |> Multi.run(:send_dog_treat, &send_dog_treat(user))
  end

  defp retrieve_customer(val, customer_id, token) when is_nil(customer_id) do
    register_customer(val, token["email"], token["id"])
  end
  defp retrieve_customer(_, customer_id, _) do
    Stripe.Customer.retrieve(customer_id)
  end
  # ...

```

@[1-3, 11-16]

@[1-16]

Note: Ecto.Multi is a data structure for grouping multiple Repo operations. functions include "insert", "update" & delete in addition to 'run'.  Changesets checked for these.  This is very useful when an operation depends on the value of a previous operation. For this reason, the function given as callback to run/3 and run/5 will receive all changes performed by the multi so far as a map in the first argument.  The function given to run must return {:ok, value} or {:error, value} as its result.

---

### Call the function

```elixir
    charge = Repo.transaction(
      PaymentService.manage_stripe_charge(
        user, design_id, design_name, token, retail)
    )
```

---

### Returns

<p> {:ok, %{return_values}}
  <br />
  OR
  <br />
  {:error, failed_operation, failed_value, changes_so_far}
</p>

Note:  The multi map is an accumulator ... which brings us to our final topic:  Recursion

---
## Recursion

#### the best thing since fritos
---

```elixir
  defp check_treats(treats_list) do
    _check_treats(treats_list, %{total_count: 0, types: []})
  end

  defp _check_treats([], info_map), do: info_map
  defp _check_treats([head | tail], info_map) do
    treat_info = get_treat_info(head)

    _check_treats(tail,
      %{total_count: treat_info.count + info_map.total_count,
        types: [treat_info | info_map.types]
      })
  end

  defp get_treat_info(treat) do
    %{count: treat.count, name: treat.name}
  end
```

@[1-3]

@[5]

@[6-13]

@[1-17]

Note: Linked List data structure, in place of iteration over while loops in Ruby.

---

<img src="assets/dog-sign-thanks.jpg" width="75%" />

---

## Links

---

#### Floki

##### https://github.com/philss/floki


#### Ecto.Multi

##### https://hexdocs.pm/ecto/Ecto.Multi.html

---

#### Stripity Stripe

##### https://hexdocs.pm/stripity_stripe/2.0.0-alpha.10


#### Ticket to Fly

##### https://gitpitch.com/cathyzoller/ticket_to_fly

---

## Acknowledgements


#### https://www.freepik.com/free-photos-vectors/dog

#### http://moderndogmagazine.com/articles/dog-sweaters-so-cute-youll-want-wear-them/91180
