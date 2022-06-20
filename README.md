# NAME

Object::Destroyer - Make objects with circular references DESTROY normally

# SYNOPSIS

    use Object::Destroyer;

    ## Use a standalone destroyer to release something
    ## when it falls out of scope
    BLOCK:
    {
        my $tree = HTML::TreeBuilder->new_from_file('somefile.html');
        my $sentry = Object::Destroyer->new( $tree, 'delete' );
        ## Here you can safely die, return, call last BLOCK or next BLOCK.
        ## The tree will be deleted automatically
    }

    ## Use it to break circular references
    {
        my $var;
        $var = \$var;
        my $sentry =  Object::Destroyer->new( sub {undef $var} );
        ## No more memory leaks!
        ## $var will be released when $sentry leaves the block
    }

    ## Destroyer can be used as a nearly transparent wrapper
    ## that will pass on method calls normally.
    {
        my $Mess = Big::Custy::Mess->new;
        print $Mess->hello;
    }

    package Big::Crusty::Mess;
    sub new {
        my $self = bless {}, shift;
        $self->populate;
        return Object::Destroyer->new( $self, 'release' );
    }
    sub hello { "Hello World!" }
    sub release { ...actual code to clean-up the memory... }

# DESCRIPTION

One of the biggest problem with working with large, nested object trees is
implementing a way for a child node to see its parent. The easiest way to
do this is to add a reference to the child back to its parent.

This results in a "circular" reference, where A refers to B refers to A.
Unfortunately, the garbage collector perl uses during runtime is not capable
of knowing whether or not something ELSE is referring to these circular
references.

In practical terms, this means that object trees in lexically scoped
variable ( e.g. `my $Object = Tree->new` ) will not be cleaned up when
they fall out of scope, like normal variables. This results in a memory leak
for the life of the process, which is a bad thing when using mod\_perl or
other processes that live for a long time.

Object::Destroyer allows for the creation of "Destroy" handles. The handle is
"attached" to the circular relationship, but is not a part of it. When the
destroy handle falls out of scope, it will be cleaned up correctly, and while
being cleaned up, it will also force the data structure it is attached to to be
destroyed as well.
Object::Destroyer can call a specified release method on an object
(or method `DESTROY` by default).
Alternatively, it can execute an arbitrary user code passed to constructor as a
code reference.

## Use as a Standalone Handle

The simplest way to use the class is to create a standalone destroyer,
preferably in the same lexical content. ( i.e. immediately after creating
the object to be destroyed)

    sub plagiarise {
      # Parse in a large nested document
      my $filename = shift;
      my $document = My::XML::Tree->open($filename);

      # Create the Object::Destroyer to clean it up as needed
      my $sentry = Object::Destroyer->new( $document, 'release' );

      # Continue with the Document as normal
      if ($document->author == $me) {
          # Normally this would have leaked the document
          return new Error("You already own the Document");
      }

      $document->change_author($me);
      $document->save;

      # We don't have to $Document->DESTROY here
      return 1;
    }

When the `$sentry` falls out of scope at the end of the sub, it will force
the cirularly linked `$Document` to be cleaned up at the same time, rather
than being forced to manually call `$Document-<gt`release> at each and every
location that the sub could possible return.

Using the Object::Destroyer object to force garbage collection to work
properly allows you to neatly sidestep the inadequecies of the perl garbage
collector and work the way you normally would, even with big objects.

## Use to clean-up data structures

If a data structure with circular refereces has no method to release memory,
you can create an `Object::Destroyer` object that will do the job.
Pass a code reference (most probably created by an anonymous subrotine block)
to the constructor of the sentry object, and this code will be called
upon leaving the scope.

    {
       $params{other}        = \%other_params;
       $other_params{params} = \%params;

       my $sentry = Object::Destroyer->new( sub {undef $params{other}} );
       ##
       ## From now on, memory of %params will be
       ## safely released when block is exited.
       ##

       ... code with return, next or last ...

    }

## Use as a Transparent Wrapper

For situations where a class is always going to produce circular references,
you may wish to build this improved clean up directly into the class itself,
and with a few exceptions everything will just work the same.

Take the following example class

    package My::Tree;

    use strict;
    use Object::Destroyer;

    sub new {
        my $self = bless {}, shift;
        $self->init; ## assume that circular references are made

        ## Return the Object::Destroyer, with ourself inside it
        my $wrapper = Object::Destroyer->new( $self, 'release' );
        return $wrapper;
    }

    sub release {
      my $self = shift;
      foreach (values %$self) {
          $_->DESTROY if ref $_ eq 'My::Tree::Node';
      }
      %$self = ();
    }

We might use the class in something like this

    sub process_file {
        # Create a new tree
        my $tree = My::Tree->new( source => shift );

        # Process the Tree
        if ($tree->comments) {
            $tree->remove_comments or return;
        }
        else {
            return 1; # Nothing to do
        }

        my $filename = $tree->param('target') or return;
        $tree->write($filename) or return;

        return 1;
    }

We were able to work with the data, and at no point did we know that we were
working with a Object::Destroyer object, rather than the My::Tree object itself.

## Resource Usage

To implement the transparency, there is a slight CPU penalty when a method is
called on the wrapper to allow it to pass the method through to the encased
object correctly, and without appearing in the `caller()` information. Once the
method is called on the underlying object, you can make further method calls
with no penalty and access the internals of the object normally.

## Problems with Wrappers and ref or UNIVERSAL::isa

Although it may ACT exactly like what's inside it, is isn't really it. Calling
`ref $wrapper` or `blessed $wrapper` will return `'Object::Destroyer'`, and
not the class of the object inside it.

Likewise, calling `UNIVERSAL::isa( $wrapper, 'My::Tree' )` or
`UNIVERSAL::can( $wrapper, 'param' )` directly as functions will also not work.
The two alternatives to this are to either use `$Wrapper->isa` or
`$wrapper->can`, which will be caught and treated normally, or simple
don't use a wrapper and just use the standalone cleaners.

# METHODS

- new

        my $sentry = Object::Destroyer->new( $object );
        my $sentry = Object::Destroyer->new( $object, 'method_name' );
        my $sentry = Object::Destroyer->new( $code_reference );

    The `new` constructor takes as arguments either a single blessed object with
    an optional name of the method to be called, or a refernce to code to be executed.
    If the method name is not specified, the `DESTROY` method is assumed.
    The constructor will die if the object passed to it does not have the specified method.

- DESTROY

        $sentry->DESTROY;
        undef $sentry;

    You may explicitly `DESTROY` the Destroyer at any time you wish.
    This will also `DESTROY` the encased object at the same. This can allow for
    legacy cases relating to Wrappers, where a user expects to have to manually
    `DESTROY` an object even though it is not needed. The `DESTROY` call will be
    accepted and dealt with as it is called on the encased object.

- dismiss

        $sentry->dismiss;

    If you have changed your mind and you don't want Destroyer object to do
    its job, dismiss it. You may continue to use it as a wrapper, though.

# SEE ALSO

Another option for dealing with circular references are _weak references_
(stable since Perl 5.8.0, see [Scalar::Util](https://metacpan.org/pod/Scalar%3A%3AUtil)). See also [GTop::Mem](https://metacpan.org/pod/GTop%3A%3AMem)
and [Devel::Monitor](https://metacpan.org/pod/Devel%3A%3AMonitor) for monitoring memory leaks.
The latter module contains a discussion on object desing with weak references.

For lexically scoped resource management, see also [Scope::Guard](https://metacpan.org/pod/Scope%3A%3AGuard),
[Sub::ScopeFinalizer](https://metacpan.org/pod/Sub%3A%3AScopeFinalizer) and [Hook::Scope](https://metacpan.org/pod/Hook%3A%3AScope).

# KNOWN ISSUES

There is a compatibility issue with [Test::MockObject::Extends](https://metacpan.org/pod/Test%3A%3AMockObject%3A%3AExtends). You cannot
extend an object wrapped by Object::Destroyer because our custom `can` method
needs to be called on an instance, but Test::MockObject::Extends calls it on
the class, and will error.

# SUPPORT

Bugs and other issues should be reported via GitHub at

[https://github.com/simbabque/Object-Destroyer/issues](https://github.com/simbabque/Object-Destroyer/issues).

# AUTHORS

- Adam Kennedy <adamk@cpan.org>
- Igor Gariev <gariev@hotmail.com>
- Julien Fiegehenn <simbabque@cpan.org>

# COPYRIGHT

Copyright 2004 - 2022 Adam Kennedy.

This program is free software; you can redistribute
it and/or modify it under the same terms as Perl itself.

The full text of the license can be found in the
LICENSE file included with this module.
