---
layout: post
title:  "Using dry-system in a Rails application"
date:   2017-03-07 19:00:00 -0300
lang: en
categories: ruby dry-rb dry-system dry-container dry-anto_inject rails
description: Setup dry-system in Rails
---
It was some months ago when I first heard of dry-rb. In case you don't know, **dry-rb** is a set of Ruby libraries (or gems). They claim to be a *collection of next-generation Ruby libraries*. Each gem provides a single piece of functionality and they build on top op each other. They are pretty cool and I highly recommend for you to take a look at [them](http://dry-rb.org/). I also want to thanks my co-worker [Gabriel Malaquias](https://github.com/GabrielMalakias) for presenting me them.

Back to the subject, I want to talk about **dry-system**. It's a superset of the **dry-container** gem, which provide registering and resolving of components, a IoC container. On top of that, **dry-system** also provides some cool features like automatically registration of components based on file paths and they can be lazily registered or eagerly registered. **dry-system** also uses **dry-auto_inject**, providing automatically injection of components into your classes. They make a perfect match.

To demonstrate how to use **dry-system**, I created a simple Rails 5 app called NextGenStudents. It contains a single Student model, aswell a controller, views and a few specs. There's a **services** directory inside the app directory, where our components classes are. The code is available on [GitHub](https://github.com/gmmoreira). So, let's begin.

## Setup
### Gemfile
Let's begin by adding dry-system in the Gemfile. dry-system depends on the dry-container and dry-auto_inject gems, so you get them for free.

```ruby
gem 'dry-system'
```
And then just run `bundle install`.

### Initialization
Next step is to create and configure our dry-system container. For this, I created a initializer `container.rb` in the `config/initializers`. You can name it whatever you want, as long as it make sense to you. In the initializer, we will require the necessary files, create and configure our container.

```ruby
require 'dry/system/container'

class AppContainer < Dry::System::Container
  configure do |config|
    config.root = (Pathname.pwd + 'app')

    config.auto_register = %w[ services ]
  end

  load_paths!('services')
end

AppImport = AppContainer.injector

AppContainer.finalize! if Rails.env == 'production'
```
We begin by requiring the necessary files and then we create a class, inheriting from the `Dry::System::Container`. Now, let's take a deeper look in the configuration. In my opinion, it's the hardest part.

This is tree of the app directory. I removed some dirs and files to get more to the point.

```bash
.
├── app
│   ├── controllers
│   │   ├── application_controller.rb
│   │   ├── concerns
│   │   └── students_controller.rb
│   ├── models
│   │   ├── application_record.rb
│   │   ├── concerns
│   │   └── student.rb
│   ├── services
│   │   ├── all_students.rb
│   │   ├── concerns
│   │   │   └── service_concern.rb
│   │   ├── create_student.rb
│   │   ├── find_student.rb
│   │   ├── send_email.rb
│   │   └── welcome_student.rb
```

First, we need to setup the root directory. By default, it's setup to `Pathname.pwd`, equivalent to the `Rails.root` method. The container can only auto-register components that are below the root dir. This means that the container would be able to find components in the app directory, lib directory, etc... The root directory also affects how the identifier of each component will be. In my case, the components I want are located in the `app/services` directory, so I setup the root directory to `Pathname.pwd + 'app'`.

Next, we got `auto_register` property. We need to assign an Array of directories name, relative to the root, from which we want to components to be looked up to. I only added the 'services' entry. These directories also affects how the identifiers of the components will be generated.

Last, we setup the `load_paths!` method. This modifies the `$LOAD_PATH` global variable and it's very important to correctly, as the container will require the found components. Without proper setup, it will fail to require them.

With this setup, I get the following components auto-registered: `["create_student", "welcome_student", "send_email", "concerns.service_concern", "find_student", "all_students"]`.

Now a question. Couldn't I just leave root to default (pwd) and just setup `auto_register and `load_paths!` method? Yes, but as I said, both affects the components identifier generation. If I was to use it like this:

```ruby
class AppContainer < Dry::System::Container
  configure do |config|
    config.auto_register = %w[ app/services ]
  end

  load_paths!('app')
end
```
I would ended up the following identifiers:
```ruby
["services.create_student",
 "services.welcome_student",
 "services.send_email",
 "services.concerns.service_concern",
 "services.find_student",
 "services.all_students"]
```
In my case all the components are located in the `app/services` directory and I didn't wanted to a services prefix in the identifiers, that's why I setup the paths different.

After declaring the container class, we assign the AppImport constant a container injector. This is an instance of the Dry::AutoInject class from dry-auto_inject gem and it's backed by our container object. By using this Injector, we will be able to do dependency injection anywhere in our project.

Lastly, we can `finalize!` the container. This will freeze the container for modifications and will eager load everything. That's why I added the `Rails.env == 'production'`, as this should only be used in production environments.

## Dependency Injection
Now that our container, components and injector are good to go, we can start using them. I will be using them into my StudentsController class. Please notice I am using dry-monads, which are pretty cool too, but they are subject to another post.

```ruby
class StudentsController < ApplicationController
  include AppImport['create_student', 'welcome_student', 'all_students', 'find_student']
  include Dry::Monads::Either::Mixin

  def index
    @students = all_students.call.or do |students|
      flash[:alert] = 'Could not load all students. Please check with your system administrator.'
      Left(students)
    end.value
  end

  def new
    @student = Student.new
  end

  def show
    find_student.call(params[:id]).fmap do |student|
      @student = student
    end.or_fmap do
      flash[:error] = 'Ocurred an error when loading student.'
      redirect_to students_url
    end
  end

  def create
    create_student.call(student_params).bind do |student|
      welcome_student.call(student).fmap do
        flash[:notice] = 'Student created succesfully.'
        redirect_to student_url(id: student.id)
      end.or_fmap do
        flash[:alert] = 'Student created but failed to welcome him.'
        redirect_to student_url(id: student.id)
      end
    end.or do |student|
      flash[:error] = 'Failed to create student.'
      @student = student
      render :new, status: :conflict
    end
  end

  private

  def student_params
    params.require(:student).permit(*%i[first_name last_name email])
  end
end
```
The previously defined AppImport was used to include the required components in the class. Basically it will create an attribute reader for each component. Out of curiosity, in the insides it will also modify the `self.new` and `initialize` methods, so componets will be injected when the object is being initialized.
With components included, then it's just a matter of calling the attibute reader method and make use of the component. In my case, all the services responds to `call` method.
