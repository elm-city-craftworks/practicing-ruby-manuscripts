As a Ruby programmer, you almost certainly make use of inheritance-based object
modeling on a daily basis. In fact, extending base classes and mixing
modules into your objects may be so common for you that you rarely 
need to think about the mechanics involved in doing so. If you are like 
most Ruby programmers, your readiness to apply this complex design paradigm 
throughout your projects is both a blessing and a curse.

On the one hand, your ability to make good use of inheritance-based modeling
without thinking about its complexity is a sign that it works well as an
abstraction. But on the other hand, having this familiar tool constantly within
reach makes it harder to recognize alternative approaches that may lead to
greater simplicity in certain contexts. Because no one tool is a golden hammer,
it is a good idea to understand the limitations of your preferred modeling
techniques as well as their virtues.

In this article, I will guide you through three properties of
inheritance-based modeling that can lead to design complications unless they are
given careful consideration. These are meant to be starting points for
conversation more-so than tutorials on what to do and what not to do, so please
attempt some of the homework exercises I've included at the bottom of the
article!

### PROBLEM 1: There is no encapsulation along ancestry chains

Inheritance-based modeling is most commonly used for behavior sharing, but 
what it actually provides is implementation sharing. Among other things,
this means that no matter how many ancestors an object has, all of its 
methods and state end up getting defined in a single namespace. If you 
aren't careful, this lack of encapsulation between objects in an 
inheritance relationship can easily bite you.

To test your understanding of this problem, see if you can spot the bug in the
following example: 

```ruby
require "prawn"

class StyledDocument < Prawn::Document
  def style(params)
    @font_name = params[:font]
    @font_size = params[:size]
  end

  def styled_text(content)
    font(@font_name) do
      text(content, :size => @font_size)
    end
  end
end

StyledDocument.generate("example.pdf") do 
  text "This is the default font size and face"

  style(:font => "Courier", :size => 20)

  styled_text "This line should be in size 20 Courier"

  text "This line should be in the default font size and face"
end
```

This example runs without raising any sort of explicit error, but produces
the following incorrect output:

![](http://i.imgur.com/xjOpU.png)

There aren't a whole lot of things that can go wrong in this example,
and so you have probably figured out the source of the problem by now:
`StyledDocument` and `Prawn::Document` each define `@font_size`, but 
they each use it for a completely different purpose. As a result, calling 
`StyledDocument#style` triggers a side effect that leads to this
subtle defect.

To verify that a naming collision to blame for this problem, you can 
try renaming the `@font_size` variable in `StyledDocument` to 
something else, such as `@styled_font_size`. Making that tiny 
change will cause the example to produce the correct output, 
as shown below:

![](http://i.imgur.com/1O23U.png)

However, this is only a superficial fix, and does not address the root problem.
The real issue is that without true subobjects with isolated state, the chance
of clashing with a variable used by an ancestor increases as your
ancestry chain grows. If you look at the mixins that `ActiveRecord::Base` 
depends on, you'll find examples of a 
[scary lack of encapsulation](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/transactions.rb#L327-345)
that will make you wonder how things don't break more often.

To make matters worse, the lack of encapsulation between objects in an
inheritance relationship also means that methods can clash in the same way that
variables can. A lack of true private methods in Ruby complicates the problem
even further, because there simply isn't a way to write a method in a parent
object that a child object can't clash with or override. One of the homework
questions for this article addresses this issue, but is worth thinking 
about for a moment before you read on.

### PROBLEM 2: Interfaces tend to grow rapidly under inheritance

I am going to attempt a proof without words for this particular problem, and
leave it up to you to figure out *why* this can be a source of maintenance
headaches, but please share your thoughts in the comments:

```ruby
>> (ActiveRecord::Base.instance_methods | 
    ActiveRecord::Base.private_instance_methods)
=> [:logger, :configurations, :default_timezone, :schema_format,
:timestamped_migrations, :init_with, :initialize_dup, :encode_with, :==, :eql?,
:hash, :freeze, :frozen?, :<=>, :readonly?, :readonly!, :inspect, :to_yaml,
:yaml_initialize, :_attr_readonly, :_attr_readonly?, :primary_key_prefix_type,
:table_name_prefix, :table_name_prefix?, :table_name_suffix,
:table_name_suffix?, :pluralize_table_names, :pluralize_table_names?,
:store_full_sti_class, :store_full_sti_class?, :store_full_sti_class=,
:default_scopes, :default_scopes?, :_accessible_attributes,
:_accessible_attributes?, :_accessible_attributes=, :_protected_attributes,
:_protected_attributes?, :_protected_attributes=, :_active_authorizer,
:_active_authorizer?, :_active_authorizer=, :_mass_assignment_sanitizer,
:_mass_assignment_sanitizer?, :_mass_assignment_sanitizer=, :validation_context,
:validation_context=, :_validate_callbacks, :_validate_callbacks?,
:_validate_callbacks=, :_validators, :_validators?, :_validators=,
:lock_optimistically, :attribute_method_matchers, :attribute_method_matchers?,
:attribute_types_cached_by_default, :time_zone_aware_attributes,
:skip_time_zone_conversion_for_attributes,
:skip_time_zone_conversion_for_attributes?, :partial_updates, :partial_updates?,
:partial_updates=, :serialized_attributes, :serialized_attributes?,
:serialized_attributes=, :[], :[]=, :record_timestamps, :record_timestamps?,
:record_timestamps=, :_validation_callbacks, :_validation_callbacks?,
:_validation_callbacks=, :_initialize_callbacks, :_initialize_callbacks?,
:_initialize_callbacks=, :_find_callbacks, :_find_callbacks?, :_find_callbacks=,
:_touch_callbacks, :_touch_callbacks?, :_touch_callbacks=, :_save_callbacks,
:_save_callbacks?, :_save_callbacks=, :_create_callbacks, :_create_callbacks?,
:_create_callbacks=, :_update_callbacks, :_update_callbacks?,
:_update_callbacks=, :_destroy_callbacks, :_destroy_callbacks?,
:_destroy_callbacks=, :auto_explain_threshold_in_seconds,
:auto_explain_threshold_in_seconds?, :nested_attributes_options,
:nested_attributes_options?, :include_root_in_json, :include_root_in_json?,
:include_root_in_json=, :reflections, :reflections?, :reflections=,
:_commit_callbacks, :_commit_callbacks?, :_commit_callbacks=,
:_rollback_callbacks, :_rollback_callbacks?, :_rollback_callbacks=,
:connection_handler, :connection_handler?, :connection,
:clear_aggregation_cache, :transaction, :destroy, :save, :save!,
:rollback_active_record_state!, :committed!, :rolledback!, :add_to_transaction,
:with_transaction_returning_status, :remember_transaction_record_state,
:clear_transaction_record_state, :restore_transaction_record_state,
:transaction_record_state, :transaction_include_action?, :serializable_hash,
:to_xml, :from_xml, :as_json, :from_json, :read_attribute_for_serialization,
:reload, :mark_for_destruction, :marked_for_destruction?,
:changed_for_autosave?, :_destroy, :reinit_with, :clear_association_cache,
:association_cache, :association, :run_validations!, :touch, :_attribute,
:type_cast_attribute_for_write, :read_attribute_before_type_cast, :changed?,
:changed, :changes, :previous_changes, :changed_attributes, :to_key, :id, :id=,
:id?, :query_attribute, :attributes_before_type_cast, :raw_write_attribute,
:read_attribute, :method_missing, :attribute_missing, :respond_to?,
:has_attribute?, :attribute_names, :attributes, :attribute_for_inspect,
:attribute_present?, :column_for_attribute, :clone_attributes,
:clone_attribute_value, :arel_attributes_values, :attribute_method?,
:respond_to_without_attributes?, :locking_enabled?, :lock!, :with_lock, :valid?,
:perform_validations, :validates_acceptance_of, :validates_confirmation_of,
:validates_exclusion_of, :validates_format_of, :validates_inclusion_of,
:validates_length_of, :validates_size_of, :validates_numericality_of,
:validates_presence_of, :errors, :invalid?, :read_attribute_for_validation,
:validates_with, :run_callbacks, :to_model, :to_param, :to_partial_path,
:attributes=, :assign_attributes, :mass_assignment_options,
:mass_assignment_role, :sanitize_for_mass_assignment,
:mass_assignment_authorizer, :cache_key, :quoted_id,
:populate_with_current_scope_attributes, :new_record?, :destroyed?, :persisted?,
:delete, :becomes, :update_attribute, :update_column, :update_attributes,
:update_attributes!, :increment, :increment!, :decrement, :decrement!, :toggle,
:toggle!, :psych_to_yaml, :to_yaml_properties, :in?, :blank?, :present?,
:presence, :acts_like?, :try, :duplicable?, :to_json, :instance_values,
:instance_variable_names, :require_or_load, :require_dependency,
:require_association, :load_dependency, :load, :require, :unloadable, :nil?,
:===, :=~, :!~, :class, :singleton_class, :clone, :dup, :initialize_clone,
:taint, :tainted?, :untaint, :untrust, :untrusted?, :trust, :to_s, :methods,
:singleton_methods, :protected_methods, :private_methods, :public_methods,
:instance_variables, :instance_variable_get, :instance_variable_set,
:instance_variable_defined?, :instance_of?, :kind_of?, :is_a?, :tap, :send,
:public_send, :respond_to_missing?, :extend, :display, :method, :public_method,
:define_singleton_method, :object_id, :to_enum, :enum_for, :psych_y,
:class_eval, :silence_warnings, :enable_warnings, :with_warnings,
:silence_stderr, :silence_stream, :suppress, :capture, :silence, :quietly,
:equal?, :!, :!=, :instance_eval, :instance_exec, :__send__, :__id__,
:initialize, :to_ary, :_run_validate_callbacks, :_run_validation_callbacks,
:_run_initialize_callbacks, :_run_find_callbacks, :_run_touch_callbacks,
:_run_save_callbacks, :_run_create_callbacks, :_run_update_callbacks,
:_run_destroy_callbacks, :_run_commit_callbacks, :_run_rollback_callbacks,
:serializable_add_includes, :associated_records_to_validate_or_save,
:nested_records_changed_for_autosave?, :validate_single_association,
:validate_collection_association, :association_valid?,
:before_save_collection_association, :save_collection_association,
:save_has_one_association, :save_belongs_to_association,
:assign_nested_attributes_for_one_to_one_association,
:assign_nested_attributes_for_collection_association,
:assign_to_or_mark_for_destruction, :has_destroy_flag?, :reject_new_record?,
:call_reject_if, :raise_nested_attributes_record_not_found, :unassignable_keys,
:association_instance_get, :association_instance_set, :create_or_update,
:create, :update, :notify_observers, :should_record_timestamps?,
:timestamp_attributes_for_create_in_model,
:timestamp_attributes_for_update_in_model, :all_timestamp_attributes_in_model,
:timestamp_attributes_for_update, :timestamp_attributes_for_create,
:all_timestamp_attributes, :current_time_from_proper_timezone,
:clear_timestamp_attributes, :write_attribute, :field_changed?,
:clone_with_time_zone_conversion_attribute?, :attribute_changed?,
:attribute_change, :attribute_was, :attribute_will_change!, :reset_attribute!,
:attribute?, :attribute_before_type_cast, :attribute=,
:convert_number_column_value, :attribute, :match_attribute_method?,
:missing_attribute, :increment_lock, :_merge_attributes, :halted_callback_hook,
:assign_multiparameter_attributes, :instantiate_time_object,
:execute_callstack_for_multiparameter_attributes, :read_value_from_parameter,
:read_time_parameter_value, :read_date_parameter_value,
:read_other_parameter_value, :extract_max_param_for_multiparameter_attributes,
:extract_callstack_for_multiparameter_attributes, :type_cast_attribute_value,
:find_parameter_position, :quote_value, :ensure_proper_type,
:destroy_associations, :default_src_encoding, :irb_binding, :Digest,
:initialize_copy, :remove_instance_variable, :sprintf, :format, :Integer,
:Float, :String, :Array, :warn, :raise, :fail, :global_variables, :__method__,
:__callee__, :eval, :local_variables, :iterator?, :block_given?, :catch, :throw,
:loop, :caller, :trace_var, :untrace_var, :at_exit, :syscall, :open, :printf,
:print, :putc, :puts, :gets, :readline, :select, :readlines, :`, :p, :test,
:srand, :rand, :trap, :exec, :fork, :exit!, :system, :spawn, :sleep, :exit,
:abort, :require_relative, :autoload, :autoload?, :proc, :lambda, :binding,
:set_trace_func, :Rational, :Complex, :gem, :gem_original_require, :BigDecimal,
:y, :Pathname, :j, :jj, :JSON, :singleton_method_added,
:singleton_method_removed, :singleton_method_undefined]
```

There is a specific issue I have with interface explosion, and it isn't so much
to do with code organization as it is with state management. Can you 
guess what my concern is?

### PROBLEM 3: Balancing reuse and customization can be tricky

Some ancestors provide methods that are designed to be replaced by 
their descendants. When executed well, this pattern provides a convenient
balance between code reuse and customization. However, because it is 
impossible to account for all possible customizations that descendants 
of a base object will want to make, this approach has its
limitations.

This design problem is best explained by example, and you can find a great 
one in Ruby itself. Start by considering the following trivial code, paying
particular attention to its output:

```ruby
class Person
  def initialize(name, email)
    @name  = name
    @email = email
  end
end

person = Person.new("Gregory Brown", "gregory@practicingruby.com")

p person    #=~
#<Person:0x0000010108bbf8 @name="Gregory Brown", 
#                         @email="gregory@practicingruby.com">

puts person #=~
#<Person:0x0000010108bbf8>     
```

Under the hood, `p` calls `person.inspect`, and `puts` calls
`person.to_s`. What you see above is output from the default implementation 
of each of those methods. Arguably, `Object#inspect` 
provides useful debugging output, but `Object#to_s` is
more of a template method that needs to be overridden in order to be useful. The
following code shows how easy to customize things by simply adding your own `to_s`
definition:

```ruby
class Person
  def initialize(name, email)
    @name  = name
    @email = email
  end

  def to_s
    "#{@name} <#{@email}>"
  end
end

person = Person.new("Gregory Brown", "gregory@practicingruby.com")

puts person #=~ Gregory Brown <gregory@practicingruby.com>  
```

On the surface, there is nothing wrong with this code: this is exactly what a
template-method based extension mechanism should look like. However, due to the
weird way that `Object#inspect` works in Ruby 1.9, defining your own `to_s`
implementation has some unpleasant side effects that are likely to surprise you:

```ruby
p person #=~ Gregory Brown <gregory@practicingruby.com>  
```

If you look at the [definition of
Object#inspect](https://github.com/ruby/ruby/blob/trunk/object.c#L486-511),
you'll find that this behavior is by design. In a nutshell, the method is set
up to provide its default output if `to_s` has not been overridden, but simply
delegate to `to_s` if it has been. This is problematic, because `to_s` is meant
to be used for humanized output such as what you saw in the previous
example, not debugging output.

The unfortunate consequence of this problem is that if you define `to_s` in your
objects, you must also define a meaningful `inspect`, and if you want to 
reproduce the same behavior as `Object#inspect`, you need to implement 
it yourself. While this is mostly a problem of brittle code and it is not
specifically related to inheritance, the problem is compounded by
inheritance-based modeling. For example, suppose the `Person` class was defined
as shown above, and you decided to subclass it:

```ruby
class Employee < Person 
  def initialize(name, email, role)
    super(name, email)
    @role = role
  end
end
```

If `Person` does define its own `inspect` method, `Employee` will inherit the
same problem. On the other hand, if `Person` does implement `inspect`, it needs
to take care to implement it in a way that's suitably general to account for
what its descendants might find useful. This invites the same design challenges
that caused this problem in the first place, which means that `Employee` may end
up cleaning up after its parent object in a similar way. Unfortunately,
brittleness tends to cascade downwards throughout ancestry chains.

### Homework exercises 

This article is on the short-side, and it also leaves out a lot of the story
from each of these points. I did this intentionally to encourage you to
participate in an active discussion on this topic. To get the most out of this
article, please complete at least one of the following homework exercises:

1) Show a realistic example of an accidental method naming collision, in a
similar spirit to the state-based example shown in Problem #1. For bonus
points, choose an example that involves private methods.

2) Post a comment in response to the "interface explosion" example shown 
in Problem #2. You can either try to guess what my main concern about 
it is, or share your own concerns. If instead you feel that there is 
nothing wrong with this kind of design, explain why you think that.

3) Come up with another downside of inheritance-based modeling, and provide an
example of it. If you have trouble coming up with your own, you may want to look
into issues that can arise from overriding methods, or perhaps explore what
happens when you mix traditional inheritance-based modeling with
`method_missing`.

4) Share an example of a library or project which is difficult to work with
because of the way it uses inheritance-based modeling, or describe problems you've run
into with your own projects due to inheritance.

5) Share the conventions and guidelines you follow to avoid the problems
described in this article, as well as other problems you've encountered with
inheritance-based modeling.

Looking forward to seeing your responses! Don't worry about getting the *right*
answers, discussion threads here on Practicing Ruby are about learning, not
necessarily showing off what you already know.
