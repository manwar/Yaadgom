# NAME

Yaadgom - Yet Another Automatic Document Generator (On Markdown)

# SYNOPSIS

    use Yaadgom;

    # create an instance
    my $foo = Yaadgom->new;

    # call this method each request you want to document
    $foo->process_response(
        folder => 'test', # what 'folder' or 'category' this is
        weight => 1     , # default order
        req    => HTTP::Request->new ( GET => 'http://www.example.com/foobar' ),
        res    => HTTP::Response->new( 200, 'OK', [ 'content-type' => 'application/json' ], '{"ok":1}' ),
    );

    # iterate over processed document, for each file.
    # NOTE: This does not write to any file.
    $foo->map_results(
        sub {
            my (%info) = @_;

            is( $info{file},   'foobar', '"foobar" file' );
            is( $info{folder}, 'test',   '"test" folder' );
            ok( $info{str}, 'has str' );
        }
    );

# DESCRIPTION

Yaadgom helps you document your requests (to create an API Reference or something like that).

Yaadgom output string in markdown format, so you can use those generated files on http://daux.io or github

For each time you call "process\_response" it will generate a new section composed of:

    ## Title with $desc
    defined $file_name ? <small>$file_name</small>
    exists $opt{extra}{name} ? > $opt{extra}{name}
    ### Request
    <pre> &format_body( $req->as_string ) </pre>
    ### Response
    <pre> &format_body( $res->as_string ) </pre>

# METHODS

## new

    Yaadgom->new(
        # add file_name on the generated document fragment, if you can pass undef to disable this feature
        file_name => "$0",

        # in case you want to do something when this objects destroy, like call ->export_to_dir
        on_destroy => sub { .. },
    );

## process\_response

    $self->process_response(
        folder => 'General',
        weight => -150, # set as "first" thing on document
        req    => HTTP::Request->new ( GET => 'http://www.example.com/login' ),
        res    => HTTP::Response->new( 200, 'OK', [ 'content-type' => 'application/json' ], '{"has_password":1}' ),
        extra => {
             fields => { has_password => ['the user has password', 'but can came from facebook']},
             you_can_extend_using => { 'Class_Trigger' => 'to process something else' }
        }
    );

## map\_results

    iterate over processed document, for each file.

    $self->map_results(
        sub {
            my (%info) = @_;

        }
    );

## export\_to\_dir

    # note that this do an append operation on files, so you may reset / truncate your directory before calling this.
    # this is done because you may want multiple tests writing to same file, in different moments.
    $self->export_to_dir(
        dir => '/tmp/
    );

# Class::Trigger names

On each trigger, return is used as the new version of the input. Except for \*process\_extras\*, where all return are concatenated.

Trigger / variables:

    $self0_01->call_trigger( 'filename_generated', { req => $req, file => $file } );
    $self0_01->call_trigger( 'format_title', { header => $desc } );
    $self0_01->call_trigger( 'format_body', { response_str => $body } );
    $self0_01->call_trigger( 'format_before_extras', { str => $str } );
    $self0_01->call_trigger( 'format_after_extras', { str => $str } );
    $self0_01->call_trigger( 'process_extras', %opt );
    $self0_01->call_trigger( 'format_generated_str', { str => $format_time } );

Updated @ Stash-REST 0.03

    $ grep  '$self_0_01->call_trigger' lib/Yaadgom.pm  | perl -ne '$_ =~ s/^\s+//; $_ =~ s/self-/self0_01-/; print' | sort | uniq

# Using Stash::REST for testing and writing docs at same time

Please read first [Stash::REST](https://metacpan.org/pod/Stash::REST) SYNOPSIS to understand how to use it.

Then, create some package that extends Stash::REST (you can call add\_trigger on the object of Stash::REST if you want too)

    package YourProject;

    use base qw(Stash::REST);
    use strict;

    YourProject->add_trigger( 'process_response' => \&on_process_response );

    use Yaadgom;

    my $dir = $ENV{DAUX_OUTPUT_DIR};

    # workarround for re-using same folder when Stash::REST call get and list of an created object.
    my $reuse_last_daux_top;
    my $reuse_count;

    my $instance = Yaadgom->new( on_destroy => \&_on_destroy );

    sub on_process_response {
        my ( $self, $opt ) = @_;

        my %conf = %{ $opt->{conf} };
        my $req  = $opt->{req};
        my $res  = $opt->{res};
        return if ( $opt->{res}->code != $conf{code} );
        $conf{folder} = $reuse_last_daux_top if $reuse_count;
        return unless $conf{folder};
        $reuse_count--;

        if ( $reuse_count <= 0 ) {
            $reuse_last_daux_top = $conf{folder};
            $reuse_count = exists $conf{list} ? 2 : $conf{code} == 201 ? 1 : 0;
        }

        $instance->process_response(
            req    => $req,
            res    => $res,
            folder => $conf{folder},

            extra => { %conf }
        );

    }

    sub _on_destroy {

        my $going_die = shift;
        $going_die->export_to_dir( dir => $dir );

    }

    1;

Now, after you run your script

    $obj = YourProject->new( ...)

    $obj->rest_post(
        '/zuzus',
        name  => 'add zuzu',
        list  => 1,
        folder => 'SomeFolder',
        params => [ name => 'foo', ]
    );

You should have on $ENV{DAUX\_OUTPUT\_DIR} a SomeFolder directory with zuzus.md inside.

If you copy those .md files into daux.io/docs folder, you can build something like this:

# AUTHOR

Renato CRON <rentocron@cpan.org>

# COPYRIGHT

Copyright 2015- Renato CRON

Thanks to http://eokoe.com

# LICENSE

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself.

# SEE ALSO

[Shodo](https://metacpan.org/pod/Shodo)

# SEE OTHER

[Stash::REST](https://metacpan.org/pod/Stash::REST), [Class::Trigger](https://metacpan.org/pod/Class::Trigger)
