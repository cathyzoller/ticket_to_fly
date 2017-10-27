---?image=assets/bg1.jpeg

---?image=assets/bg2.jpeg

Note: How many rails devs?  Elixir devs?  Just starting out?  Anything else you'd like me to ask?

---?image=assets/bg3.jpeg

Note: This is a practical talk about practical things, like images and functions.  Compare/Contrast (mostly contrast) with rails counterparts


---?image=assets/bg4.jpeg

Note: Let' start with images...

---?image=assets/bg5.jpeg

Note: Create thumbnails, resize etc. as with Paperclip or Carrierwave in Rails

---?image=assets/bg6.jpeg

Note: Local store (as opposed to cdn), use Arc Ecto to integrate the two. Going over the highlights here.

---

## Multiple Images

```elixir
<%= file_input @form, :image_uploads,

    class: "validate",

    multiple: true,

    name: "user[image_upload][]" %>
```

Note: From here, you can iterate over the map of files for processing or saving, either as embeds array, or association.


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

```elixir
#[debug] Processing by Knitwhiz.DesignController.update/2

Parameters: %{
  "id" => "6",
  "design" => %{
    "description" => "vest",
    "name" => "Paula's vest",
    "photo" => %Plug.Upload{
      content_type: "image/png",
      filename: "Knitting@0.5x.png",
      path: "/var/folders/2s/fs..66/T//plug-1493/multipart-53892"
    },
    "supplies" => "yarn"
  }
}
```

@[1-15]
@[8-12]

## Plug.Upload Struct

#### within changeset function params

Note: Plug.Upload  Genserver process saves upload struct to a temp directory. After process dies the file moved to permanent home (either cdn or local store)


---

# Image Gotcha

### No Record ID on Create

```elixir

# uploaders/avatar
def storage_dir(_, {_, scope}) do

   "priv/static/images/user/#{scope.id}"

end
```

Note: Scope.id not available on create.
Workaround:  Use separate changesets for create & update actions

+++

# Workaround

## User updloads images

#### separate changesets for create and update

+++

# Workaround

## Programatically Insert images


```elixir

case Repo.insert(changeset) do

  {:ok, pattern} ->

    template = Repo.get!(Template, pattern_params["template_id"])

    path = "priv/static/images/templates/#{template.id}"

    PatternImage.store({path, pattern})

    pattern_img_param = %{pattern_image_url: "../patterns/#{pattern.id}"}

    |> update_pattern

  end

```

@[1-2]

@[2-3]

@[3-4]

@[5-6]


Note: Explain Background - User form for other info -> onCreate, get image associated with parent

---

## Amazon S3

## With Arc

#### https://github.com/stavro/arc

+++

## Amazon S3

## Without Arc (Elixir apps)

* :ex_aws & :ex_aws_s3 packages
* Create a Module
* Follow the README - https://github.com/ex-aws/ex_aws

Note: use mix task to move images on s3

---

# B is for Backend

### Phoenix as an API

### Receive Image Data from a Client Application

* Send binary data
  * Arc supports binary data
* Send FormData

Note: often no need to use binary data

+++

## Form Component (React/Redux)

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

@[1-4]

@[6-11]

@[5, 10]

@[1-16]

+++

## Update Action

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
      .
      .
```

@[1-15]

@[3]

@[4]

@[6-12]

@[14]

Note: FormData objects require a POST request

+++

## FormData objects must be sent in a POST request

#### update router.ex

* resources "/designs", DesignController, except: [:new, :edit]
* post "/designs/:id", DesignController, :update

---

# C is for Copy

---

## SVG Files

### 1. Copy inside a Mix Task

#### Using File Module

```elixir

cp(source, destination, callback \\ fn _, _ -> true end)
# => {:ok} OR {:error, :reason}
```
#### OR

```elixir

copy(source, destination, bytes_count \\ :infinity)
# => {:ok, :bytes_copied} OR {:error, :reason}
```

+++

### 2. User manipulation

#### Using D3 or similar library

+++

### 3. Save the Transformed File








